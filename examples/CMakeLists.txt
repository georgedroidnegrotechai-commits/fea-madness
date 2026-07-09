# Examples subdir - placeholders for validation cases (cantilever, modal plate, thermal, etc.)
# Will be expanded with actual .cpp demos linking to fea_madness_core

add_executable(simple_demo_placeholder simple_demo.cpp)
target_link_libraries(simple_demo_placeholder PRIVATE fea_madness_core)
