Dmitriy Vetutnev, ODANT 2020


# Разработка Conan-пакетов. Методические указания.

## Общая структура пакета

## Опции пакета

### shared
### dll_sign
### with_unit_tests
### ninja

## Исходники библиотеки (и накладываемые патчи)

Исходники библиотеки располагаются в директории *src* корня проекта. Рекомендуется выкачивать их одним коммитом:

    git subtree add -P src https://github.com/library/lib.git version --squash

Про косяк гита при обновлении исходников.

    git subtree pull

Оригинальные исходники не рекомендуются менять. Свои изменения стоит вносить патчами (в будующем оно упрощает обновление версии). Не забываем добавлять файл патча в список экспорта. Для формирования патча удобно без выполнения фиксации редактировать исходники прямо в папке *src* и сразу видеть как собирается пакет. Формирование итогового патча выполняется командой ```git diff```

В файле **conanfile.py** накладывание исходников выглядит так:

    def source(self):
        tools.patch(patch_file="odant.patch")


## Сборка библиотеки
### CMake

Для интеграции с **Conan** используется следующий врапер (корень проекта, CMakeLists.txt):


    cmake_minimum_required(VERSION 3.0)
    
    include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
    conan_basic_setup()
    
    enable_testing()
    add_subdirectory(src)

При помощи его выполняется подстройка компилятора (архитектура, тип сборки) и подключение сторонних Conan-библиотек. Рекомендуемый генератор - **Ninja**. Пример метода сборки в **conanfile.py**:

    def build(self):
        build_type = "RelWithDebInfo" if self.settings.build_type == "Release" else "Debug"
        gen = "Ninja" if self.options.ninja == True else None
        cmake = CMake(self, build_type=build_type, generator=gen, msbuild_verbosity='normal')
        cmake.verbose = True
        cmake.definitions["LIB_INSTALL"] = "ON"
        cmake.definitions["LIB_DOC"] = "OFF"
        if self.options.with_unit_tests:
            cmake.definitions["LIB_TEST"] = "ON"
        cmake.configure()
        cmake.build()
        if self.options.with_unit_tests:
            if cmake.is_multi_configuration:
                self.run("ctest --output-on-failure --build-config %s" % build_type)
            else:
                self.run("ctest --output-on-failure")
        cmake.install()
        tools.rmdir(os.path.join(self.package_folder, "lib/cmake"))
        tools.rmdir(os.path.join(self.package_folder, "lib/pkgconfig"))


## Отладка сборки в локальной директории

## Упаковка продуктов

Большинство библиотек стоит упаковывать запуском инсталяции системы сборки. После инсталяции рекомендуется удалить лишние файлы из пакета (см. пример сборки).

Обязательно упаковываем скрипт интеграции с **CMake**. Без него намного сложней подключать Conan-библиотеки к сторонним проектам (приходится накладывать патчи, он сосуществовании CONAN_PKG::library они не в курсе). Так же не забываем упаковывать PDB-файлы для Visual Studio. Пример метода *package*:

### Ручная упаковка.


## Интеграция с CMake

## Тест пакета

Предназначен для тестирования корректности упаковки пакета и возможно накладываемых патчей. К тестам из комлекта библиотеки отношения не имеет. Представляют несколько исполняемых бинарников, скомпонованными с библиотеками из тестируемого пакета, и использующие одну и более функций из этих библиотек (для проверки загрузки и конфигурации библиотеки).

Файлы теста располагают в директории test_package, вложенной в корень исходных текстов пакета. Рекомендуемая система сборки - **CMake**, backend CMake - **Ninja**. Ninja предоставляется служебным Conan-пакетом. Так же этот тест можно в качестве отладочного полигона библиотеки, перед включением ее какой-либо проект. Запуск сборка и последующий запуск тестов возможны без пересборки пакета (командной **conan create**). Для этого в корне исходных пакетов выполняется команда:

    conan test test_package library/x.y.z+p@username/channel

Версию пакета, username и channel необходимо указывать полностью. При разработке тестов для облегчения свой жизни лучше избегать использования внешних зависомостей. Стоит ограничится стандартной библиотекой компилятора. Не забываем добавлять директорию **test_package/build** в исключения системы контроля версий, в ней происходит сборка тестов пакета.

Пример **conanfile.py**
    
    from conans import ConanFile, CMake
    
    class PackageTestConan(ConanFile):
        settings = "os", "compiler", "build_type", "arch"
        generators = "cmake"
        requires = "ninja/1.9.0"
    
        def imports(self):
            self.copy("*.pdb", dst="bin", src="bin")
            self.copy("*.dll", dst="bin", src="bin")
            self.copy("*.so*", dst="bin", src="lib")

        def build(self):
            cmake = CMake(self, generator="Ninja", msbuild_verbosity='normal')
            cmake.verbose = True
            cmake.configure()
        cmake.build()
        self.cmake_is_multi_configuration = cmake.is_multi_configuration
    
        def test(self):
            if self.cmake_is_multi_configuration:
                self.run("ctest --verbose --build-config %s" % self.settings.build_type)
            else:
                self.run("ctest --verbose")


Пример **CMakeLists.txt**
    
    project(PackageTest CXX)
    cmake_minimum_required(VERSION 3.0)
    
    include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
    conan_basic_setup(TARGETS)
    
    enable_testing()
    
    # library::library imported target
    find_package(library REQUIRED)
    
    add_executable(test_package test_package.cpp)
    target_link_libraries(test_package library::library)
    set_target_properties(test_package
        PROPERTIES
        INSTALL_RPATH "$ORIGIN"
        BUILD_WITH_INSTALL_RPATH True
    )
    add_test(
        NAME test_package
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/bin
        COMMAND test_package
    )
    
    # CONAN_PKG::library imported target
    
    add_executable(test_package_CONAN_PKG test_package.cpp)
    target_link_libraries(test_package_CONAN_PKG CONAN_PKG::library)
    set_target_properties(test_package_CONAN_PKG
        PROPERTIES
        INSTALL_RPATH "$ORIGIN"
        BUILD_WITH_INSTALL_RPATH True
    )
    add_test(
        NAME test_package_CONAN_PKG
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/bin
        COMMAND test_package_CONAN_PKG
    )


Для отладки теста пакета можно использовать IDE, рекомендуемая - Qt Creator. Visual Studio не подходит, т.к. перед запуском CMake (а это происходит как минимум при открытии проекта в IDE) она очищает директорию сборки и удаляет файлы интеграции с Conan (conanbuildinfo.cmake).

При использовании Qt Creator открываем CMakeLists.txt из директории test_package, директорию сборки указываем **test_package/<хэш_сборки>** (хеш сборки можно увидеть при запуске сборки теста пакета). В используемом *Kits* (*Комплект* в рускоязычном интерфейсе) должны быть указаны тот же CMake-генератор (Ninja), используемый в тесте пакета, и тот же компилятор, которым собирался пакет и его тест.


## Интеграция с CI

### conan-package-tools build.py и переменные окружения
