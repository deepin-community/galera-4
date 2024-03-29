#
# Copyright (C) 2020-2021 Codership Oy <info@codership.com>
#

add_library(galera STATIC
  mapped_buffer.cpp
  write_set.cpp
  data_set.cpp
  key_set.cpp
  write_set_ng.cpp
  trx_handle.cpp
  key_entry_os.cpp
  wsdb.cpp
  certification.cpp
  galera_service_thd.cpp
  wsrep_params.cpp
  replicator_smm_params.cpp
  gcs_action_source.cpp
  galera_info.cpp
  galera_view.cpp
  replicator.cpp
  ist.cpp
  ist_proto.cpp
  gcs_dummy.cpp
  saved_state.cpp
  replicator_smm.cpp
  replicator_str.cpp
  replicator_smm_stats.cpp
  )

target_include_directories(galera
  PRIVATE
  ${PROJECT_SOURCE_DIR}/wsrep/src
  )

# TODO: Fix.
target_compile_options(galera
  PRIVATE
  -Wno-conversion
  -Wno-unused-parameter
  )

if (GALERA_STATIC)
  target_link_libraries(galera gcs -static-libgcc)
else()
  target_link_libraries(galera gcs)
endif()

add_library(galera_smm_static STATIC
  wsrep_provider.cpp
  )

target_compile_definitions(galera_smm_static
  PRIVATE
  -DGALERA_VER="${GALERA_VERSION}"
  -DGALERA_REV="${GALERA_GIT_REVISION}"
  -DGALERA_MULTIMASTER
  )

target_include_directories(galera_smm_static
  PRIVATE
  ${PROJECT_SOURCE_DIR}/wsrep/src
  )

# TODO: Fix.
target_compile_options(galera_smm_static
  PRIVATE
  -Wno-conversion
  -Wno-unused-parameter
  )

target_link_libraries(galera_smm_static galera)

add_library(galera_smm  MODULE
  wsrep_provider.cpp
  )

set_target_properties(galera_smm PROPERTIES
  LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
  )

target_include_directories(galera_smm
  PRIVATE
  ${PROJECT_SOURCE_DIR}/wsrep/src
  )

target_compile_definitions(galera_smm
  PRIVATE
  -DGALERA_VER="${GALERA_VERSION}"
  -DGALERA_REV="${GALERA_GIT_REVISION}"
  -DGALERA_MULTIMASTER
  )

# TODO: Fix.
target_compile_options(galera_smm
  PRIVATE
  -Wno-conversion
  -Wno-unused-parameter
  )

if (GALERA_VERSION_SCRIPT)
  # Limit symbols visible from Galera DSO.
  # Doing this allows to:
  # - make the ABI more clean and concise
  # - hide symbols from commonly used libraries (boost, asio, etc.), which
  #   binds calls inside the DSO to its own versions of these libraries
  # See: https://akkadia.org/drepper/dsohowto.pdf (section 2.2.5)
  set(GALERA_LINK_OPTIONS
    -Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/galera-sym.map)
endif()

target_link_libraries(galera_smm galera ${GALERA_LINK_OPTIONS})

install(TARGETS galera_smm DESTINATION lib)

# The following checks are guaranteed to work only
# Linux platform, we skip them on others.
if (NOT ${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
  return()
endif()

#
# Post build checks:
# - Check that symbol visibility matches GALERA_VERSION_SCRIPT.
# - Check that SSL linkage matches GALERA_HAVE_SSL and GALERA_STATIC.
#
if (GALERA_VERSION_SCRIPT)
  add_custom_command(TARGET galera_smm POST_BUILD
    COMMAND
    sh -c "! ${CMAKE_OBJDUMP} -T libgalera_smm.so | grep asio 1> /dev/null"
    WORKING_DIRECTORY "${PROJECT_BINARY_DIR}"
    COMMENT "Checking library symbol visibility (hidden)"
    VERBATIM)
else()
  set(GALERA_LINK_OPTIONS "")
  add_custom_command(TARGET galera_smm POST_BUILD
    COMMAND
    sh -c "${CMAKE_OBJDUMP} -T libgalera_smm.so | grep asio 1> /dev/null"
    WORKING_DIRECTORY "${PROJECT_BINARY_DIR}"
    COMMENT "Checking library symbol visibility (not hidden)"
    VERBATIM)
endif()

if (NOT GALERA_WITH_SSL)
  message(STATUS "Building Galera without SSL")
  add_custom_command(TARGET galera_smm POST_BUILD
    COMMAND
    sh -c "! (${CMAKE_OBJDUMP} -x libgalera_smm.so | grep NEEDED.*crypto 1> /dev/null) && ! (${CMAKE_OBJDUMP} -x libgalera_smm.so | grep NEEDED.*ssl 1> /dev/null)"
    WORKING_DIRECTORY "${PROJECT_BINARY_DIR}"
    COMMENT "Verifying that library is not linked with SSL"
    VERBATIM)
else()
  if (GALERA_STATIC)
    message(STATUS "Building Galera with static SSL")
    add_custom_command(TARGET galera_smm POST_BUILD
      COMMAND
      sh -c "(${CMAKE_OBJDUMP} -t libgalera_smm.so | grep OPENSSL 1> /dev/null) && (${CMAKE_OBJDUMP} -t libgalera_smm.so | grep CRYPTO 1> /dev/null)"
      WORKING_DIRECTORY "${PROJECT_BINARY_DIR}"
      COMMENT "Verifying that library has OpenSSL linked statically"
      VERBATIM)
  else()
    message(STATUS "Building Galera with SSL")
    add_custom_command(TARGET galera_smm POST_BUILD
      COMMAND
      sh -c "(${CMAKE_OBJDUMP} -x libgalera_smm.so | grep NEEDED.*crypto 1> /dev/null) && (${CMAKE_OBJDUMP} -x libgalera_smm.so | grep NEEDED.*ssl 1> /dev/null)"
      COMMENT "Verifying that library is linked with SSL"
      WORKING_DIRECTORY "${PROJECT_BINARY_DIR}"
      VERBATIM)
  endif()
endif()
