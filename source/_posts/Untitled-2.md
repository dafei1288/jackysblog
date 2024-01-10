
```
(base) root@autodl-container-534b4e910e-ff71482e:~/autodl-tmp/envs/pkgs# cat ~/.condarc 

channels:
  - https://mirrors.ustc.edu.cn/anaconda/pkgs/main/
  - https://mirrors.ustc.edu.cn/anaconda/pkgs/free/
  - defaults
show_channel_urls: true
envs_dirs:
  - /root/autodl-tmp/envs
pkgs_dirs:
  - /root/autodl-tmp/envs/pkgs   
```

pip config set global.cache-dir /root/autodl-tmp/envs/cache


pip install -U huggingface_hub


export HF_ENDPOINT=https://hf-mirror.com


huggingface-cli download --token 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 --cache-dir /root/autodl-tmp/envs/hfcache --resume-download --local-dir-use-symlinks False runwayml/stable-diffusion-v1-5 --local-dir pretrained_models/stable-diffusion-v1-5

huggingface-cli download --token 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 --cache-dir /root/autodl-tmp/envs/hfcache --resume-download --local-dir-use-symlinks False stabilityai/sd-vae-ft-mse --local-dir pretrained_models/sd-vae-ft-mse


huggingface-cli download --token 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 --cache-dir /root/autodl-tmp/envs/hfcache --resume-download --local-dir-use-symlinks False zcxu-eric/MagicAnimate --local-dir pretrained_models/MagicAnimate



python -m demo.gradio_animate