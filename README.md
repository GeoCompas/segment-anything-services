# Running the Segment Anything Encoder and Decoder as Services

This project contains two seperate Torchserve services, one for the encoder (best run on a GPU) and the decoder (CPU). Check out the [original repo](https://github.com/facebookresearch/segment-anything) for more info.

## Setup

### 1. Downloading model weights

If you have access, download from the devseed s3:

```
aws s3 sync model-weights s3://segment-anything/model-weights/
```

otherwise, get checkpoints from the original repo: https://github.com/facebookresearch/segment-anything/tree/main#model-checkpoints

### 2a. Package the torch weights for GPU encoding

This step takes a long time presumably because the uncompiled weights are massive. Packaging the ONNX model is faster in the later steps.

```
mkdir -p model_store_encode
torch-model-archiver --model-name sam_vit_h_encode --version 1.0.0 --serialized-file model-weights/sam_vit_h_4b8939.pth --handler handler_encode.py 
mv sam_vit_h_encode.mar model_store_encode/sam_vit_h_encode.mar
```

### 2b. Exporting the ONNX model for CPU decoding

```
mkdir -p models                    
python scripts/export_onnx_model.py --checkpoint model-weights/sam_vit_h_4b8939.pth --model-type vit_h --output models/sam_vit_h_decode.onnx
```

### 2c. Package the ONNX model for CPU decoding with the handler

We'll put this in the model_store_decode directory, to keep the onnx model files distinct from the torchserve .mar model archives. model_store/ is created automatically by Torchserve in the container, which is why we're make a local folder here called "model_store_decode".

```
mkdir -p model_store_decode
torch-model-archiver --model-name sam_vit_h_decode --version 1.0.0 --serialized-file models/sam_vit_h_decode.onnx --handler handler_decode.py
mv sam_vit_h_decode.mar model_store_decode/sam_vit_h_decode.mar
```


### 3a. Building the gpu torchserve container for image encoding
With the GPU, inference time should be about 1.8 seconds or less depending on the GPU. On an older 1080 Ti Pascal GPU, inference time is 1.67 seconds without compilation.

```
docker build -t torchserve-sam-gpu .
bash start_serve_encode_gpu.sh $(pwd)/model_store_encode
```

### 3b. Building the cpu torchserve container for image decoding

```
docker build -t torchserve-sam-cpu -f Dockerfile-cpu .
bash start_serve_decode_cpu.sh $(pwd)/model_store_decode
```

### 4. Building jupyter server container

Use this container to test the model in a GPU enabled jupyter notebook server with geospatial and pytorch dependencies installed.

```
docker build -t sam-dev -f Dockerfile-dev .
```

### 5. Test the endpoints

You can run `test_endpoint.ipynb` to then use the two running services you started above.

### 5. Run jupyter server container

This is a GPU enabled container that is set up with SAM and some other dependencies we commonly use. You can use it to try out SAM model in a notebook environment. Remove the `--gpus` arg if you don't have a GPU.

```
docker run -it --rm \
    -v $HOME/.aws:/root/.aws \
    -v "$(pwd)":/segment-anything-services \
    -p 8888:8888 \
    -e AWS_PROFILE=devseed \
    --gpus all sam-dev
```

### (Potentially) Frequently Asked Questions
Q: Why two services?

A: We're exploring cost effective ways to run image encoding in a separate, on-demand way from the CPU decoder. Eventually we'd like to remove the need for the CPU torserve on the backend and run the decoding in the browser.

Q: Can I contribute or ask questions?
A: This is currently more of a "working in the open" type repo that we'd like to share with others, rather than a maintained project. But feel free to open an issue if you have an idea. Please understand if we don't respond or are slow to respond.

## License

The model and code is licensed under the [Apache 2.0 license](LICENSE).

## References

Kirillov, A., Mintun, E., Ravi, N., Mao, H., Rolland, C., Gustafson, L., ... Girshick, R. (2023). Segment Anything. *arXiv:2304.02643*. https://github.com/facebookresearch/segment-anything

The scripts/export_onnx_model.ipynb and notebooks/sam_onnx_model_example_fox.ipynb are from the original repo.