llama_add_compile_flags()

#
# libraries
#

# llama

file(GLOB LLAMA_MODELS_SOURCES "models/*.cpp")

add_library(llama
            ../include/llama.h
            llama.cpp
            llama-adapter.cpp
            llama-arch.cpp
            llama-batch.cpp
            llama-chat.cpp
            llama-context.cpp
            llama-cparams.cpp
            llama-grammar.cpp
            llama-graph.cpp
            llama-hparams.cpp
            llama-impl.cpp
            llama-io.cpp
            llama-kv-cache.cpp
            llama-kv-cache-iswa.cpp
            llama-kv-cache-dsa.cpp
            llama-memory.cpp
            llama-memory-hybrid.cpp
            llama-memory-hybrid-iswa.cpp
            llama-memory-recurrent.cpp
            llama-mmap.cpp
            llama-model-loader.cpp
            llama-model-saver.cpp
            llama-model.cpp
            llama-quant.cpp
            llama-sampler.cpp
            llama-vocab.cpp
            unicode-data.cpp
            unicode.cpp
            unicode.h
            ${LLAMA_MODELS_SOURCES}
            )

set_target_properties(llama PROPERTIES
    VERSION ${LLAMA_INSTALL_VERSION}
    SOVERSION 0
    MACHO_CURRENT_VERSION 0 # keep macOS linker from seeing oversized version number
)

target_include_directories(llama PRIVATE .)
target_include_directories(llama PUBLIC ../include)
target_compile_features   (llama PRIVATE cxx_std_17) # don't bump

target_link_libraries(llama PUBLIC ggml)

# MoE expert-offload driver lives in the ggml-cuda target (see
# ggml/src/ggml-cuda/CMakeLists.txt). llama only needs the header for the
# build_moe_ffn seam; the symbols resolve via ggml-cuda -> ggml.
target_include_directories(llama PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/../../../moe-driver/include)

# ...but the real driver is ONLY compiled into ggml-cuda. On non-CUDA builds
# (CPU-only inference, or any non-CUDA backend) those symbols don't exist, yet
# the seam (build_moe_ffn) + the host daemon's moe_driver_init reference them
# unconditionally. Compile inert host stubs INTO libllama so the same archive
# that needs the symbols also defines them (no link-order fragility). MoE offload
# is CUDA-only, so the stubs just report "inactive" and the seam runs stock.
if (NOT GGML_CUDA)
    target_sources(llama PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/../../../moe-driver/src/moe_stub.cpp)
endif()

if (BUILD_SHARED_LIBS)
    set_target_properties(llama PROPERTIES POSITION_INDEPENDENT_CODE ON)
    target_compile_definitions(llama PRIVATE LLAMA_BUILD)
    target_compile_definitions(llama PUBLIC  LLAMA_SHARED)
endif()
