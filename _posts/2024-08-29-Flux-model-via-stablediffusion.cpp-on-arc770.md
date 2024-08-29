## clone github repository

```sh
git clone --recursive https://github.com/leejet/stable-diffusion.cpp
cd stable-diffusion.cpp
```

### Build project with SYCL enabled

```sh
export ONEAPI_ROOT=/home/martin/intel/oneapi/
# Export relevant ENV variables
source /home/martin/intel/oneapi/setvars.sh
# Option 1: Use FP32 (recommended for better performance in most cases)
cmake . -DSD_SYCL=ON -DCMAKE_C_COMPILER=icx -DCMAKE_CXX_COMPILER=icpx -DGGML_CCACHE=OFF
cmake --build . --config Release
```

### Download flux model files

```sh
mkdir models
cd models

# Download flux-dev (quantized to 4-Bit)
wget https://huggingface.co/lllyasviel/FLUX.1-dev-gguf/resolve/4d9780f1dc9abff77202c13d30d7af44f67b0ea4/flux1-dev-Q4_0.gguf?download=true -O flux1-dev-Q4_0.gguf

# Download vae ()
wget https://huggingface.co/black-forest-labs/FLUX.1-dev/resolve/main/ae.safetensors -O ae.safetensors

# Download clip_l
wget https://huggingface.co/comfyanonymous/flux_text_encoders/resolve/main/clip_l.safetensors -O clip_l.safetensors

# Download t5xxl
wget https://huggingface.co/comfyanonymous/flux_text_encoders/resolve/main/t5xxl_fp16.safetensors -O t5xxl_fp16.safetensors
```

### Run model
```sh
./bin/sd --diffusion-model  ./models/flux1-dev-Q4_0.gguf --vae ./models/ae.safetensors --clip_l ./models/clip_l.safetensors --t5xxl ./models/t5xxl_fp16.safetensors  -p "Create a hyper-realistic image of a futuristic, scary robot face inspired by Angelina Jolie. The robot's metallic face has sleek, angular features, glowing blue eyes, and exposed circuitry beneath synthetic skin that mirrors Jolie's iconic high cheekbones and full lips. The face is emotionless yet menacing, with one side damaged to reveal wires and mechanical components beneath. Her well-known piercing gaze is now robotic and cold, with a glowing internal light. The background is dark and dystopian, with dim neon lights casting an eerie glow on the metallic surface, amplifying the robot's terrifying and futuristic presence." --cfg-scale 1.0 --sampling-method euler -v
```

### Result
![alt text](./assets/angelina.png "Title")