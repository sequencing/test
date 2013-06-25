################################################################################
##
## Isaac Genome Alignment Software
## Copyright (c) 2010-2012 Illumina, Inc.
##
## This software is provided under the terms and conditions of the
## Illumina Open Source Software License 1.
##
## You should have received a copy of the Illumina Open Source
## Software License 1 along with this program. If not, see
## <https://github.com/sequencing/licenses/>.
##
## The distribution includes the code libraries listed below in the
## 'redist' sub-directory. These are distributed according to the
## licensing terms governing each library.
##
################################################################################
##
## file CMakeLists.txt
##
## Configuration file for the manual subfolder
##
## author Roman Petrovski
##
################################################################################
include ("${iSAAC_GLOBALS_CMAKE}")

include(UseMarkdown)

file(GLOB_RECURSE iSAAC_MD_SOURCES *.md)

add_markdown_target("manual" "${CMAKE_CURRENT_BINARY_DIR}" "${iSAAC_MD_SOURCES}")

install_markdown_target("manual")
