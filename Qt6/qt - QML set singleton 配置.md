QML set singleton 配置：

```Cmake
set(SINGLETON_QML_FILES
    APICredentials.qml
)

set_source_files_properties(
    ${SINGLETON_QML_FILES}
    PROPERTIES
        QT_QML_SINGLETON_TYPE TRUE
)

qt_add_executable(appQNetworkDemo
    main.cpp
)

qt_add_qml_module(appQNetworkDemo
    URI QNetworkDemo
    QML_FILES
        Main.qml
        SOURCES backendhelper.h backendhelper.cpp
        QML_FILES APICredentials.qml
)
```

