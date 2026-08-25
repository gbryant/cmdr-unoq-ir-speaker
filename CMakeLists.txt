cmake_minimum_required(VERSION 3.20.0)

# Pull commander before Zephyr initializes — download ONLY (don't process commander's own
# Pico CMakeLists). SOURCE_SUBDIR=include has no CMakeLists.txt, so MakeAvailable populates
# the source but skips add_subdirectory (the non-deprecated way to do FetchContent_Populate).
# GIT_TAG pins a commander release — move it with `cmdr pin <tag>`.
include(FetchContent)
FetchContent_Declare(commander GIT_REPOSITORY https://github.com/gbryant/commander.git GIT_TAG v1.1 SOURCE_SUBDIR include)
add_compile_definitions(MAX_COMMANDS=8)
FetchContent_MakeAvailable(commander)
set(COMMANDER ${commander_SOURCE_DIR})

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(cmdr-unoq-ir-speaker)

# The Uno Q's two brains talk over the multiplexed channel bus (ch0 console + pub/sub),
# not a plain UART console. The runner switches transport on this flag.
add_compile_definitions(COMMANDER_ENABLE_CHANNELS)

target_include_directories(app PRIVATE ${COMMANDER} ${COMMANDER}/include)
target_sources(app PRIVATE
    src/main.cpp
    ${COMMANDER}/runners/zephyr/runner.cpp
    ${COMMANDER}/core/CommandRegistry.cpp
    ${COMMANDER}/transport/channels/ChannelBusRunner.cpp
    ${COMMANDER}/platform/zephyr/ZephyrIRModule.cpp   # dead-stripped unless `ir` is enabled
    ${COMMANDER}/hal/zephyr/hal.cpp
)
