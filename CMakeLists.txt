cmake_minimum_required (VERSION 3.29)

project (VulkanTutorialRoot)

if(MSVC)
    set(CMAKE_VS_DEBUGGER_WORKING_DIRECTORY "${CMAKE_CURRENT_LIST_DIR}/attachments")
endif()

add_subdirectory(attachments)
add_subdirectory(attachments/simple_engine)

# 6. Target-specific adjustments after they are created.
if(TARGET SimpleEngine)
    if(MSVC)
        # On Windows, we set the working directory to a subdirectory of simple_engine
        # so that "../Assets" resolves correctly to "attachments/simple_engine/Assets".
        set_target_properties(SimpleEngine PROPERTIES
            VS_DEBUGGER_WORKING_DIRECTORY "${CMAKE_CURRENT_LIST_DIR}/attachments/simple_engine/build")
    endif()
endif()