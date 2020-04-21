Dmitriy Vetutnev, ODANT 2020


# Разработка Conan-пакетов. Методические указания.

## Общая структура пакета

## Опции пакета

### shared
### dll_sign
### with_unit_tests
### ninja

## Исходники библиотеки (и накладываемые патчи)

## Сборка библиотеки

## Отладка сборки в локальной директории

## Упаковка продуктов

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


## Интеграция с CI