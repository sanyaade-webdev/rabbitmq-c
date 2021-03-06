project(librabbitmq "C")

set(CMAKE_INCLUDE_CURRENT_DIR ON)

# Stuff dealing with code generation
set(AMQP_CODEGEN_PY "${CMAKE_CURRENT_BINARY_DIR}/amqp_codegen.py")
set(CODEGEN_PY "${CMAKE_CURRENT_BINARY_DIR}/codegen.bld.py")
set(AMQP_SPEC_JSON_PATH "${AMQP_CODEGEN_DIR}/amqp-rabbitmq-0.9.1.json")

#generate amqp_framing.h/amqp_framing.c
add_custom_command(
  OUTPUT ${CODEGEN_PY}
  COMMAND ${CMAKE_COMMAND} ARGS -E copy ${CMAKE_CURRENT_SOURCE_DIR}/codegen.py ${CODEGEN_PY}
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/codegen.py
  VERBATIM)

add_custom_command(
  OUTPUT ${AMQP_CODEGEN_PY}
  COMMAND ${CMAKE_COMMAND} ARGS -E copy ${AMQP_CODEGEN_DIR}/amqp_codegen.py ${AMQP_CODEGEN_PY}
  DEPENDS ${AMQP_CODEGEN_DIR}/amqp_codegen.py ${AMQP_CODEGEN_TARGET}
  VERBATIM)

add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/amqp_framing.h
  COMMAND ${PYTHON_EXECUTABLE} ARGS ${CODEGEN_PY} header ${AMQP_SPEC_JSON_PATH} ${CMAKE_CURRENT_BINARY_DIR}/amqp_framing.h
  DEPENDS ${AMQP_SPEC_JSON_PATH} ${CODEGEN_PY} ${AMQP_CODEGEN_PY}
  VERBATIM)

add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/amqp_framing.c
  COMMAND ${PYTHON_EXECUTABLE} ARGS ${CODEGEN_PY} body ${AMQP_SPEC_JSON_PATH} ${CMAKE_CURRENT_BINARY_DIR}/amqp_framing.c
  DEPENDS ${AMQP_SPEC_JSON_PATH} ${CODEGEN_PY} ${AMQP_CODEGEN_PY}
  VERBATIM)


SET(CONFIG_CONTENTS "#define VERSION \"0.0.1\"
#ifndef __cplusplus
# define inline ${C_INLINE}
#endif // __cplusplus
")

#prepare config.h
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/config.h" ${CONFIG_CONTENTS})

if(WIN32)
  set(SOCKET_IMPL "windows")
else(WIN32)
  set(SOCKET_IMPL "unix")
endif(WIN32)

if(MSVC)
  set(MSINTTYPES_INCLUDE "${CMAKE_CURRENT_SOURCE_DIR}/../msinttypes")
endif(MSVC)

include_directories(${SOCKET_IMPL} ${MSINTTYPES_INCLUDE})
add_definitions(-DBUILDING_LIBRABBITMQ)

set(RABBITMQ_SOURCES
    ${CMAKE_CURRENT_BINARY_DIR}/amqp_framing.h
    ${CMAKE_CURRENT_BINARY_DIR}/amqp_framing.c
    amqp_api.c  amqp.h 
    amqp_connection.c  amqp_mem.c  amqp_private.h  amqp_socket.c  amqp_table.c
    amqp_url.c
    ${SOCKET_IMPL}/socket.h ${SOCKET_IMPL}/socket.c
)

add_library(rabbitmq SHARED ${RABBITMQ_SOURCES})

if(WIN32)
  target_link_libraries(rabbitmq ws2_32)
endif(WIN32)

install(TARGETS rabbitmq
	RUNTIME DESTINATION bin
	LIBRARY DESTINATION lib
	ARCHIVE DESTINATION lib
	PUBLIC_HEADER DESTINATION include)

install(FILES amqp.h ${CMAKE_CURRENT_BINARY_DIR}/amqp_framing.h
	DESTINATION include)


