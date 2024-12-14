function(clickhouse_import_crate)
    # This is a workaround for Corrosion case sensitive build type matching in
    # _generator_add_cargo_targets(), that leads to different paths in
    # IMPORTED_LOCATION and real path of the library.
    #
    # It uses CMAKE_CONFIGURATION_TYPES and $<CONFIG>, so here we preserve the
    # case of ${CMAKE_BUILD_TYPE} in ${CMAKE_CONFIGURATION_TYPES}.
    if ("${CMAKE_BUILD_TYPE_UC}" STREQUAL "DEBUG")
        set(CMAKE_CONFIGURATION_TYPES "${CMAKE_BUILD_TYPE};release")
    else()
        set(CMAKE_CONFIGURATION_TYPES "${CMAKE_BUILD_TYPE};debug")
    endif()

    if (CMAKE_BUILD_TYPE_UC STREQUAL "DEBUG")
        set(profile "")
    else()
        if (ENABLE_THINLTO)
            set(profile "release-thinlto")
        else()
            set(profile "release")
        endif()
    endif()

    corrosion_import_crate(
        ${ARGN}
        CRATES _ch_rust_skim_rust;_ch_rust_prql
        NO_STD
        PROFILE ${profile}
        LOCKED
        FLAGS --offline
    )

endfunction()

# -Wno-dollar-in-identifier-extension: cxx bridge complies names with '$'
# -Wno-unused-macros: unused CXXBRIDGE1_RUST_STRING
set(CXXBRIDGE_CXXFLAGS "-Wno-dollar-in-identifier-extension -Wno-unused-macros")
set(RUST_CXXFLAGS "${RUST_CXXFLAGS} ${CXXBRIDGE_CXXFLAGS}")
message(STATUS "RUST_CXXFLAGS (for skim): ${RUST_CXXFLAGS}")

configure_file(".cargo/config.toml.in" ".cargo/config.toml" @ONLY)

clickhouse_import_crate(MANIFEST_PATH Cargo.toml)

if (SANITIZE STREQUAL "memory")
    # For more info: https://doc.rust-lang.org/beta/unstable-book/compiler-flags/sanitizer.html#memorysanitizer
    corrosion_set_cargo_flags(
        _ch_rust_prql
        "-Zbuild-std=core,alloc,std,panic_abort"
    )
    corrosion_set_cargo_flags(
        _ch_rust_skim_rust
        "-Zbuild-std=core,alloc,std,panic_abort"
    )
    list(APPEND RUSTFLAGS "-Zsanitizer=memory" "-Zsanitizer-memory-track-origins")
endif()

corrosion_import_crate(
    MANIFEST_PATH Cargo.toml
    CRATES chcache
    PROFILE release
    LOCKED
    FLAGS --offline
)
corrosion_set_env_vars(
    chcache
    RUSTFLAGS=
    RUST_CFLAGS=
    RUST_CXXFLAGS=
    CFLAGS=
    CXXFLAGS=
)
corrosion_set_hostbuild(chcache)

target_include_directories(_ch_rust_prql INTERFACE prql/include)
add_library(ch_rust::prql ALIAS _ch_rust_prql)

target_include_directories(_ch_rust_skim_rust INTERFACE skim/include)
add_library(ch_rust::skim ALIAS _ch_rust_skim_rust)

