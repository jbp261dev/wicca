# Copyright 2011-2012,2014,2017 Free Software Foundation, Inc.
#
# This file is part of Wicca project
#
########################################################################
#Set the project name
project (Wicca_tools)

# Add a library with the above sources
add_library(${PROJECT_NAME} SHARED
${PROJECT_SOURCE_DIR}/src/logging.cpp)
add_library(sub::tools ALIAS ${PROJECT_NAME})

target_include_directories( ${PROJECT_NAME}
    PUBLIC ${PROJECT_SOURCE_DIR}/include
)