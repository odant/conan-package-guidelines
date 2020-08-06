Dmitriy Vetutnev, ODANT 2020


# Разработка Conan-пакетов. Методические указания.

## Общая структура пакета

## Поля пакета

## Опции пакета

Ниже приведены типичные опции пакета. Некоторые из них могут отсутствовать.

### shared

Тип библиотеки динамическая/статическая. Build-хелпер для CMake по ней автоматически настраивает сборку.

### fPIC

Генерация позиционно-независимого кода. Актуально для UNIX-платформ, на Windows эта опция удаляется. При сборке динамической библиотеки опция fPIC должна быть всегда включена. Build-хелпер для CMake по ней устанавливает переменную **CMAKE_POSITION_INDEPENDENT_CODE** Отключать имеет смысл только для сборки полностью статических бираников под очень мелкие платформы (где даже Линукса нет). Значение по-умолчанию - **True**

### dll_sign

Подписать бинарные артефакты. Только для Windows и динамических библиотек. Для UNIX-платформ и статических библиотек удалена. Значение по-умолчанию - **True**. Подробней подпись бинарников описана в разделе *Упаковка*

Для Windows-машин без приватных сертификатов (которыми и должны быть все рабочие станции) рекомендется прописать настройку в **~/.conan/profiles/default**:

    [options]
    *:dll_sign=False

### with_unit_tests

Собрать и запустить unit-тесты из комплекта библиотеки. Значение по-умолчанию - **True**. Опция не должна менять ID бинарной сборки пакета (собственно бинарники в пакете от тестов не зависят):

    def package_id(self):
        self.info.options.with_unit_tests = "any"

Подробней о тестах описано в разделе *Сборка*

### ninja

Собрать пакет при помощи **Ninja**. Значение по-умолчанию - **True**

Пример настройки опций в **conanfile.py**:

    def configure(self):
        # MT(d) static library
        if self.settings.os == "Windows" and self.settings.compiler == "Visual Studio":
            if self.settings.compiler.runtime == "MT" or self.settings.compiler.runtime == "MTd":
                self.options.shared=False
        if self.settings.os != "Windows" or self.options.shared == False:
            del self.options.dll_sign
        if self.settings.os == "Windows":
            del self.options.fPIC

## Исходники библиотеки (и накладываемые патчи)

Исходники библиотеки располагаются в директории *src* корня проекта. Рекомендуется выкачивать их одним коммитом:

    git subtree add -P src https://github.com/library/lib.git v1.2.3 --squash

*v1.2.3* - это тег коммита. Вместо него так же можно указывать и ID коммита.

Обновление исходников библиотеки осуществляется такой командой:

    git subtree pull -P src https://github.com/library/lib.git v2.3.4 --squash

В современных (на 2020 год) версиях Git команда *git subtree pull* почему-то падает приблизительно с такой ошибкой:

    $ git subtree pull -P src https://github.com/nodejs/node.git v12.18.3 --squash
    From https://github.com/nodejs/node
     * tag                       v12.18.3   -> FETCH_HEAD
    fatal: ambiguous argument '3e896191a4d9078f0412bc03cdf80761ff2636ef^0': unknown revision or path not in the working tree.
    Use '--' to separate paths from revisions, like this:
    'git <command> [<revision>...] -- [<file>...]'
    could not rev-parse split hash 3e896191a4d9078f0412bc03cdf80761ff2636ef from commit c8d6993ca60986e18ebe1004587621bbc02265b9
    Can't squash-merge: 'src' was never added.

Эта бага обходится следующим способом. Добавляем к локальному репозиторию URL репозитория библиотеки и загружаем (но не применяем) историю в недра Git.

    $ git remote add src_upstream https://github.com/nodejs/node.git
    $ git fetch src_upstream
    remote: Enumerating objects: 9877, done.
    remote: Counting objects: 100% (9877/9877), done.
    remote: Total 19666 (delta 9876), reused 9877 (delta 9876), pack-reused 9789
    Receiving objects: 100% (19666/19666), 25.06 MiB | 4.89 MiB/s, done.
    Resolving deltas: 100% (15250/15250), completed with 4142 local objects.
    From https://github.com/nodejs/node
     * [new branch]              archived-io.js-v0.10 -> src_upstream/archived-io.js-v0.10
     * [new branch]              archived-io.js-v0.12 -> src_upstream/archived-io.js-v0.12
     * [new branch]              canary-base          -> src_upstream/canary-base
     * [new branch]              initial-quic-part-1  -> src_upstream/initial-quic-part-1
     * [new branch]              initial-quic-part-2  -> src_upstream/initial-quic-part-2
     * [new branch]              initial-quic-part-3  -> src_upstream/initial-quic-part-3
     * [new branch]              v11.x                -> src_upstream/v11.x
     * [new branch]              v12.x                -> src_upstream/v12.x
     * [new branch]              v12.x-staging        -> src_upstream/v12.x-staging
     * [new branch]              v13.x                -> src_upstream/v13.x
     * [new branch]              v13.x-staging        -> src_upstream/v13.x-staging
     * [new branch]              v14.6.0-proposal     -> src_upstream/v14.6.0-proposal
     * [new branch]              v14.x                -> src_upstream/v14.x
     * [new branch]              v14.x-staging        -> src_upstream/v14.x-staging
     * [new tag]                 v10.22.0             -> v10.22.0
     * [new tag]                 v12.18.2             -> v12.18.2
     * [new tag]                 v12.18.3             -> v12.18.3
     * [new tag]                 v14.6.0              -> v14.6.0
     * [new tag]                 v14.5.0              -> v14.5.0

После этого команда *git subtree pull* выполняется без ошибок.

Оригинальные исходники не рекомендуются вносить изменения (после этого команда *git subtree pull* перестает работать). Свои изменения стоит вносить патчами (в будующем оно упрощает обновление версии). Не забываем добавлять файл патча в список экспорта. Для формирования патча удобно без выполнения фиксации редактировать исходники прямо в папке *src* и сразу видеть как собирается пакет. Формирование итогового патча выполняется командой ```git diff```

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

    def build_requirements(self):
        if self.options.ninja:
            self.build_requires("ninja/1.9.0")

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

## Встроенные unit-тесты библиотеки.

## Упаковка продуктов

Большинство библиотек стоит упаковывать запуском инсталяции системы сборки. После инсталяции рекомендуется удалить лишние файлы из пакета (см. пример сборки).

Обязательно упаковываем скрипт интеграции с **CMake**. Без него намного сложней подключать Conan-библиотеки к сторонним проектам (приходится накладывать патчи, он сосуществовании CONAN_PKG::library они не в курсе). Так же не забываем упаковывать PDB-файлы для Visual Studio. Пример метода *package*:

### Ручная упаковка.

### Подпись упакованых артефактов (Windows-only)

Про метод package и модуль windows_signtool

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
        build_requires = "ninja/1.9.0"
    
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

При использовании Qt Creator открываем CMakeLists.txt из директории test_package, директорию сборки указываем **test_package/build/<хэш_сборки>** (хеш сборки можно увидеть при запуске сборки теста пакета). В используемом *Kits* (*Комплект* в рускоязычном интерфейсе) должны быть указаны тот же CMake-генератор (Ninja), используемый в тесте пакета, и тот же компилятор, которым собирался пакет и его тест.


## Интеграция с CI

### conan-package-tools build.py и переменные окружения
