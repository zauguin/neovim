file( GLOB NEOVIM_SOURCES *.c )

foreach(sfile ${NEOVIM_SOURCES})
  get_filename_component(f ${sfile} NAME)
  if(${f} MATCHES "^(regexp_nfa.c|farsi.c|arabic.c)$")
    list(APPEND to_remove ${sfile})
  endif()
endforeach()

list(REMOVE_ITEM NEOVIM_SOURCES ${to_remove})
list(APPEND NEOVIM_SOURCES "${PROJECT_BINARY_DIR}/config/auto/pathdef.c")

file( GLOB OS_SOURCES os/*.c )

if(CMAKE_C_COMPILER_ID MATCHES "Clang")
  if(DEFINED ENV{SANITIZE})
    message(STATUS "Enabling the sanitizers")
    add_definitions(-DEXITFREE) # is this necessary for LeakSanitizer?
    add_definitions(-fno-sanitize-recover -fno-omit-frame-pointer
      -fno-optimize-sibling-calls -fsanitize=address -fsanitize=undefined)
    set(CMAKE_EXE_LINKER_FLAGS
      "-fsanitize=address -fsanitize=undefined ${CMAKE_EXE_LINKER_FLAGS}")
    set(CMAKE_SHARED_LINKER_FLAGS
      "-fsanitize=address -fsanitize=undefined ${CMAKE_SHARED_LINKER_FLAGS}")
  endif()
endif()

if(NOT DEFINED ENV{SKIP_EXEC})
  add_executable (nvim ${NEOVIM_SOURCES} ${OS_SOURCES})
endif()
if(NOT DEFINED ENV{SKIP_UNITTEST})
  add_library (nvim-test MODULE ${NEOVIM_SOURCES} ${OS_SOURCES})
endif()

# The libraries we link against for nvim
set(NVIM_LINK_LIBRARIES m ${LibUV_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})

# Add any libraries needed for a specific platform
if(HAVE_CLOCK_GETTIME)
  # Work around libuv.a not linking in rt.
  list(APPEND NVIM_LINK_LIBRARIES rt)
endif(HAVE_CLOCK_GETTIME)

if (LibIntl_FOUND)
  list(APPEND NVIM_LINK_LIBRARIES ${LibIntl_LIBRARY})
endif()

if(NOT DEFINED ENV{SKIP_EXEC})
  target_link_libraries (nvim ${NVIM_LINK_LIBRARIES})
endif()
if(NOT DEFINED ENV{SKIP_UNITTEST})
  target_link_libraries (nvim-test ${NVIM_LINK_LIBRARIES})
endif()

include(CheckLibraryExists)
check_library_exists(curses tgetent "" HAVE_LIBCURSES)
if (HAVE_LIBCURSES)
  if(NOT DEFINED ENV{SKIP_EXEC})
    target_link_libraries(nvim curses)
  endif()
  if(NOT DEFINED ENV{SKIP_UNITTEST})
    target_link_libraries(nvim-test curses)
  endif()
else()
  find_package(Curses REQUIRED)
  if(NOT DEFINED ENV{SKIP_EXEC})
    target_link_libraries(nvim ${CURSES_LIBRARIES})
  endif()
  if(DEFINED ENV{SKIP_UNITTEST})
    target_link_libraries(nvim-test ${CURSES_LIBRARIES})
  endif()
endif()

include_directories ("${PROJECT_SOURCE_DIR}/src/proto") 

if(NOT DEFINED ENV{SKIP_EXEC})
  install(TARGETS nvim RUNTIME DESTINATION bin)
endif()
