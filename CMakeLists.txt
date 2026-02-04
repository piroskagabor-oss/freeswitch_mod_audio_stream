cmake_minimum_required(VERSION 3.15)
project(mod_audio_stream)

if(NOT DEFINED FREESWITCH_INCLUDE_DIR)
  message(FATAL_ERROR "Please set FREESWITCH_INCLUDE_DIR")
endif()

if(NOT DEFINED FREESWITCH_CORE_LIB)
  message(FATAL_ERROR "Please set FREESWITCH_CORE_LIB")
endif()

# Module source files
set(SOURCES
  mod_audio_stream.c
  audio_streamer_glue.cpp
  base64.cpp
  buffer/ringbuffer.c
  # add other .c/.cpp files from the repo as needed
)

# Target
add_library(mod_audio_stream SHARED ${SOURCES})

# Include FreeSWITCH headers
target_include_directories(mod_audio_stream PRIVATE "${FREESWITCH_INCLUDE_DIR}")

# Link against freeswitchcore import library
# Create an imported target for clarity
add_library(freeswitchcore_imported STATIC IMPORTED)
set_target_properties(freeswitchcore_imported PROPERTIES
  IMPORTED_LOCATION "${FREESWITCH_CORE_LIB}"
)
target_link_libraries(mod_audio_stream PRIVATE freeswitchcore_imported)

# Ensure these are used by the target
target_include_directories(mod_audio_stream PRIVATE ${SPEEXDSP_INCLUDE_DIR})
add_library(speexdsp_imported STATIC IMPORTED)
set_target_properties(speexdsp_imported PROPERTIES IMPORTED_LOCATION "${SPEEXDSP_LIB}")
target_link_libraries(mod_audio_stream PRIVATE speexdsp_imported)
# --- end speexdsp find helper ---

find_package(ixwebsocket CONFIG REQUIRED)

target_link_libraries(mod_audio_stream PRIVATE ixwebsocket::ixwebsocket)

# Link Windows system libs (ws2_32 for sockets, crypt32 for TLS certs if needed)
target_link_libraries(mod_audio_stream PRIVATE Ws2_32 Crypt32 Bcrypt)


target_compile_definitions(mod_audio_stream PRIVATE MOD_EXPORTS=1)

# If the module uses std::filesystem or other C++17 features:
set_target_properties(mod_audio_stream PROPERTIES
  CXX_STANDARD 17
  RUNTIME_OUTPUT_DIRECTORY "${FREESWITCH_MODULES_DIR}"
)

# Post-build: copy module next to freeswitch.exe or modules dir (optional)
add_custom_command(TARGET mod_audio_stream POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_if_different
    $<TARGET_FILE:mod_audio_stream>
    "${FREESWITCH_MODULES_DIR}"
)
