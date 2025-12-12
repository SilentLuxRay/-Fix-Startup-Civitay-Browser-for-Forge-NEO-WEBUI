
## [Fix] Startup Crash on SD WebUI Forge Neo (AttributeError: 'Namespace' object has no attribute 'ckpt_dir' / 'hypernetwork_dirs')
### Repository link: https://github.com/BlafKing/sd-civitai-browser-plus
### The Issue
Stable Diffusion WebUI Forge "Neo" has updated its backend structure. Many command line arguments that define model paths (like `ckpt_dir`, `hypernetwork_dir`, `lora_dir`) have been renamed, removed, or converted into lists (e.g., `ckpt_dirs`).

The current version of CivitAI Browser+ tries to access these attributes directly via `cmd_opts.variable`, causing immediate crashes (AttributeError) if the specific attribute doesn't exist in the new environment.

### The Fix
We need to rewrite the directory logic to be "defensive" using `getattr`. This makes the extension compatible with **both** the new Forge Neo (which uses lists/plurals) and older WebUI versions (which use strings/singulars).

**File to edit:** `extensions\sd-civitai-browser-plus\scripts\civitai_api.py`

**Step:** Locate the `def contenttype_folder(...)` function (usually around line 25) and replace the **entire function** with the robust code below:

```python
def contenttype_folder(content_type, desc=None, fromCheck=False, custom_folder=None):
    use_LORA = getattr(opts, "use_LORA", False)
    folder = None
    if desc:
        desc = desc.upper()
    else:
        desc = "PLACEHOLDER"
    
    if custom_folder:
        main_models = custom_folder
        main_data = custom_folder
    else:
        main_models = models_path
        main_data = data_path
        
    if content_type == "modelFolder":
        folder = os.path.join(main_models)
        
    elif content_type == "Checkpoint":
        # Check priority: ckpt_dirs (list) -> ckpt_dir (str) -> Default Path
        ckpt_dirs = getattr(cmd_opts, 'ckpt_dirs', None)
        ckpt_dir = getattr(cmd_opts, 'ckpt_dir', None)
        
        if ckpt_dirs and not custom_folder:
            folder = ckpt_dirs[0] if isinstance(ckpt_dirs, list) else ckpt_dirs
        elif ckpt_dir and not custom_folder:
            folder = ckpt_dir
        else:
            folder = os.path.join(main_models, "Stable-diffusion")
            
    elif content_type == "Hypernetwork":
        # Safe check using getattr
        hyper_dir = getattr(cmd_opts, 'hypernetwork_dir', None)
        hyper_dirs = getattr(cmd_opts, 'hypernetwork_dirs', None) # Future proofing
        
        if hyper_dirs and not custom_folder:
             folder = hyper_dirs[0] if isinstance(hyper_dirs, list) else hyper_dirs
        elif hyper_dir and not custom_folder:
            folder = hyper_dir
        else:
            folder = os.path.join(main_models, "hypernetworks")
        
    elif content_type == "TextualInversion":
        embed_dir = getattr(cmd_opts, 'embeddings_dir', None)
        if embed_dir and not custom_folder:
            folder = embed_dir
        else:
            folder = os.path.join(main_data, "embeddings")
        
    elif content_type == "AestheticGradient":
        if not custom_folder:
            folder = os.path.join(extensions_dir, "stable-diffusion-webui-aesthetic-gradients", "aesthetic_embeddings")
        else:
            folder = os.path.join(custom_folder, "aesthetic_embeddings")
            
    elif content_type == "LORA":
        # Neo often uses lora_dirs (plural/list)
        lora_dirs = getattr(cmd_opts, 'lora_dirs', None)
        lora_dir = getattr(cmd_opts, 'lora_dir', None)
        
        if lora_dirs and not custom_folder:
            folder = lora_dirs[0] if isinstance(lora_dirs, list) else lora_dirs
        elif lora_dir and not custom_folder:
            folder = lora_dir
        else:
            folder = os.path.join(main_models, "Lora")
        
    elif content_type == "LoCon":
        folder = os.path.join(main_models, "LyCORIS")
        if use_LORA and not fromCheck:
            lora_dirs = getattr(cmd_opts, 'lora_dirs', None)
            lora_dir = getattr(cmd_opts, 'lora_dir', None)
            if lora_dirs and not custom_folder:
                folder = lora_dirs[0] if isinstance(lora_dirs, list) else lora_dirs
            elif lora_dir and not custom_folder:
                folder = lora_dir
            else:
                folder = os.path.join(main_models, "Lora")

    elif content_type == "DoRA":
        lora_dirs = getattr(cmd_opts, 'lora_dirs', None)
        lora_dir = getattr(cmd_opts, 'lora_dir', None)
        if lora_dirs and not custom_folder:
             folder = lora_dirs[0] if isinstance(lora_dirs, list) else lora_dirs
        elif lora_dir and not custom_folder:
             folder = lora_dir
        else:
            folder = os.path.join(main_models, "Lora")
            
    elif content_type == "VAE":
        vae_dirs = getattr(cmd_opts, 'vae_dirs', None)
        vae_path = getattr(cmd_opts, 'vae_path', None)
        
        if vae_dirs and not custom_folder:
            folder = vae_dirs[0] if isinstance(vae_dirs, list) else vae_dirs
        elif vae_path and not custom_folder:
            folder = vae_path
        else:
            folder = os.path.join(main_models, "VAE")
            
    elif content_type == "Controlnet":
        cnet_dir = getattr(cmd_opts, 'controlnet_dir', None)
        if cnet_dir and not custom_folder:
            folder = cnet_dir
        else:
            folder = os.path.join(main_models, "ControlNet")
            
    elif content_type == "Poses":
        folder = os.path.join(main_models, "Poses")
    
    elif content_type == "Upscaler":
        swinir_path = getattr(cmd_opts, 'swinir_models_path', None)
        realesrgan_path = getattr(cmd_opts, 'realesrgan_models_path', None)
        gfpgan_path = getattr(cmd_opts, 'gfpgan_models_path', None)
        bsrgan_path = getattr(cmd_opts, 'bsrgan_models_path', None)
        esrgan_path = getattr(cmd_opts, 'esrgan_models_path', None)

        if "SWINIR" in desc:
            if swinir_path and not custom_folder:
                folder = swinir_path
            else:
                folder = os.path.join(main_models, "SwinIR")
        elif "REALESRGAN" in desc:
            if realesrgan_path and not custom_folder:
                folder = realesrgan_path
            else:
                folder = os.path.join(main_models, "RealESRGAN")
        elif "GFPGAN" in desc:
            if gfpgan_path and not custom_folder:
                folder = gfpgan_path
            else:
                folder = os.path.join(main_models, "GFPGAN")
        elif "BSRGAN" in desc:
            if bsrgan_path and not custom_folder:
                folder = bsrgan_path
            else:
                folder = os.path.join(main_models, "BSRGAN")
        else:
            if esrgan_path and not custom_folder:
                folder = esrgan_path
            else:
                folder = os.path.join(main_models, "ESRGAN")
            
    elif content_type == "MotionModule":
        folder = os.path.join(extensions_dir, "sd-webui-animatediff", "model")
        
    elif content_type == "Workflows":
        folder = os.path.join(main_models, "Workflows")
        
    elif content_type == "Other":
        if "ADETAILER" in desc:
            folder = os.path.join(main_models, "adetailer")
        else:
            folder = os.path.join(main_models, "Other")
    
    elif content_type == "Wildcards":
        folder = os.path.join(extensions_dir, "UnivAICharGen", "wildcards")
        if not os.path.exists(folder):
            folder = os.path.join(extensions_dir, "sd-dynamic-prompts", "wildcards")
    
    return folder
