# ratpak ESP32 port, ESP-IDF component
#
# The upstream sources in Ratpack/ are kept as close to microsoft/calculator as
# possible, compiler warnings caused by MSVC-specific code are silenced here
# instead of changing the sources.

idf_component_register(
    SRCS
        "Ratpack/basex.cpp"
        "Ratpack/conv.cpp"
        "Ratpack/exp.cpp"
        "Ratpack/fact.cpp"
        "Ratpack/itrans.cpp"
        "Ratpack/itransh.cpp"
        "Ratpack/logic.cpp"
        "Ratpack/num.cpp"
        "Ratpack/rat.cpp"
        "Ratpack/support.cpp"
        "Ratpack/trans.cpp"
        "Ratpack/transh.cpp"
    INCLUDE_DIRS
        "."
        "Ratpack")

# -Wno-unknown-pragmas: MSVC #pragma warning in ratpak.h, PUBLIC because the
# header is compiled as part of every component that includes it
target_compile_options(${COMPONENT_LIB} PUBLIC -Wno-unknown-pragmas)
# -Wno-switch: switches over AngleType without a Radians case in trans.cpp
target_compile_options(${COMPONENT_LIB} PRIVATE -Wno-switch)
