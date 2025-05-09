# Deploying MuJoCo MPC with Docker on Windows

This guide provides step-by-step instructions to set up a Docker environment on Windows for running Google DeepMind's MuJoCo MPC (Model Predictive Control). It utilizes a CUDA-enabled Docker image with X11 forwarding for GUI applications.

## Prerequisites

Before you begin, ensure you have the following installed and configured on your Windows machine:

1.  **NVIDIA GPU:** With the latest NVIDIA drivers installed.
2.  **Docker Desktop for Windows:** Configured to use the WSL 2 backend. This is essential for GPU support in Docker.
3.  **WSL 2:** Ensure you have a WSL 2 distribution installed and integrated with Docker Desktop.
4.  **VcXsrv (or another X Server for Windows):**
    * Download and install VcXsrv from [sourceforge.net/projects/vcxsrv/](https://sourceforge.net/projects/vcxsrv/).
    * When launching VcXsrv (e.g., via XLaunch), ensure you:
        * Select "Multiple windows" or "Fullscreen".
        * Set "Display number" to `0` (or another number, but you'll need to adjust `DISPLAY` environment variable accordingly).
        * Select "Start no client".
        * **Crucially, check "Disable access control"** in the "Extra settings" page.
        * You might also need to add an inbound rule to your Windows Firewall to allow connections to VcXsrv if you encounter issues.

## Step 1: Build the Docker Image

1.  Save the following Dockerfile content into a file named `Dockerfile` in a new directory (e.g., `C:\mjpc_docker_setup\Dockerfile`):

    ```dockerfile
    # CUDA-enabled MuJoCo 3.x (Python ≥ 3.10) – CUDA ≥ 12.2
    ARG cuda_docker_tag="12.2.2-cudnn8-devel-ubuntu22.04"
    FROM nvidia/cuda:${cuda_docker_tag}

    ENV DEBIAN_FRONTEND=noninteractive \
        PYTHONUNBUFFERED=1 \
        MUJOCO_GL=egl \
        LANG=C.UTF-8
    
    RUN apt-get update && \
        apt-get install -y --no-install-recommends \
            python3 python3-pip python3-dev python3-venv \
            git wget curl build-essential \
            libosmesa6-dev libgl1-mesa-glx libglfw3 libglew-dev \
            libxrandr2 libxinerama1 libxcursor1 \
            ffmpeg vim openssh-server && \
        ln -sf /usr/bin/python3 /usr/bin/python && \
        rm -rf /var/lib/apt/lists/*

    RUN python -m pip install --upgrade --no-cache-dir pip setuptools wheel && \
        pip install --no-cache-dir \
            "mujoco>=3.3" \
            glfw \
            gymnasium[robotics] \
            && \
        pip cache purge

    RUN echo "root:123123" | chpasswd && \
        sed -i 's/#\?PermitRootLogin .*/PermitRootLogin yes/'    /etc/ssh/sshd_config && \
        sed -i 's/#\?PasswordAuthentication .*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
        sed -i 's/#\?X11Forwarding .*/X11Forwarding yes/'        /etc/ssh/sshd_config && \
        echo 'X11DisplayOffset 10'    >> /etc/ssh/sshd_config && \
        echo 'X11UseLocalhost no'     >> /etc/ssh/sshd_config

    RUN printf '#!/bin/bash\nservice ssh start\nexec "$@"\n' \
        > /usr/local/bin/docker-entrypoint && \
        chmod +x /usr/local/bin/docker-entrypoint

    ENTRYPOINT ["/usr/local/bin/docker-entrypoint"]
    CMD ["/bin/bash"]
    ```

2.  Open a terminal (PowerShell or Command Prompt) in the directory where you saved the `Dockerfile`.
3.  Build the Docker image. We'll tag it as `mujoco-py310` to match your `docker run` command:

    ```bash
    docker build -t mujoco-py310 .
    ```

## Step 2: Run the Docker Container

Once the image is built and VcXsrv is running (with display number 0 and access control disabled), run the following command in your terminal:

```bash
docker run --gpus all --env NVIDIA_DRIVER_CAPABILITIES=all -it --name mjpc_env -e DISPLAY=host.docker.internal:0.0 -v /tmp/.X11-unix:/tmp/.X11-unix mujoco-py310 /bin/bash
```

## Step 3: Inside the Container

Execute the following commands sequentially inside the Docker container's bash shell:

1. Update package lists and install build dependencies for MuJoCo MPC:

```bash
apt-get update
apt-get install -y cmake ninja-build pkg-config \
                   libeigen3-dev libgflags-dev libgoogle-glog-dev
apt-get install -y libxrandr-dev libxinerama-dev libxi-dev libxxf86vm-dev
apt-get install -y libxcursor-dev
# Note: The Dockerfile already installs libglvnd, libegl, libosmesa, libgl1-mesa, libglu1-mesa, libglfw3, and base X11 libs.
# The commands below ensure any potentially missing dev headers or specific versions needed by mjpc are present.
apt-get install -y --no-install-recommends \
                   libglvnd0 libglvnd-dev libegl1 libegl1-mesa
apt-get install -y \
                   libosmesa6-dev libgl1-mesa-dev libglu1-mesa-dev \
                   libglfw3-dev \
                   libx11-dev libxrandr-dev libxinerama-dev \
                   libxcursor-dev libxi-dev libxxf86vm-dev
```

2. Clone the `mujoco_mpc` repository:

```bash
git clone --recursive [https://github.com/google-deepmind/mujoco_mpc.git](https://github.com/google-deepmind/mujoco_mpc.git)
cd mujoco_mpc
```

3. Build `mujoco_mpc`:

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j"$(nproc)"
```

## Step 4: Testing the Setup

1. Test X11 Forwarding (GUI Test): ensure `x11-apps` are installed

```bash
apt-get update
apt-get install -y x11-apps
```

Then, run `xeyes`. A small window with a pair of eyes that follow your mouse cursor should appear on your Windows desktop. If it does, X11 forwarding is working correctly. Close the `xeyes` window.

```bash
xeyes
```

2. Test MuJoCo MPC (CPU-based Rendering with OSMesa): This test uses offscreen rendering and doesn't require a display server, but it's good for verifying the core functionality.

```bash
# cd /mujoco_mpc/build
export MUJOCO_GL=osmesa
# Ensure osmesa dev libraries are installed (they should be from previous steps or Dockerfile)
# apt-get install -y libosmesa6 libosmesa6-dev
./bin/mjpc --task cartpole
```

You should see console output from the simulation. Press `Ctrl+C` to stop it.

3. Test MuJoCo MPC (GPU-accelerated Rendering with EGL/GUI): Ensure MUJOCO_GL is set for EGL (or GLFW if you prefer and have it configured for MuJoCo's simulator).

```bash
export MUJOCO_GL=egl

# Check EGL is available
ldconfig -p | grep libEGL

# Run the cartpole task
./bin/mjpc --task cartpole
```

If your X11 forwarding (VcXsrv) and NVIDIA drivers/Docker GPU passthrough are correctly configured, you should see the MuJoCo MPC simulation window appear on your Windows desktop.

## Troubleshooting & Notes

- `host.docker.internal` not resolving? Ensure your Docker Desktop is up to date.

- ERROR: could not initialize `GLFW`:
   - `MuJoCo` itself supports headless rendering (`OSMesa` or `EGL`), but `MJPC` (MPC GUI) itself always tries to create a `GLFW` window. MJPC never compiled the `GLFW_OSMESA` path so `glfwInit()` still tries (and fails) to connect to `X11`. Check if you had supply an X server.
   - Use CPU test if it's a GPU Failure
   

- VcXsrv "Cannot open display":
  - Double-check VcXsrv is running.
  - Ensure "Disable access control" was checked when launching VcXsrv.
  - Verify the `DISPLAY` variable in the docker `run` command matches your VcXsrv display number (e.g., `:0.0` for display 0).
  - Check Windows Firewall. You might need to allow VcXsrv through the firewall.
 
- `NVIDIA-SMI has failed` or GPU issues:
  - Ensure NVIDIA drivers are correctly installed on the Windows host.
  - Ensure Docker Desktop is using the WSL 2 backend.
  - Ensure your WSL 2 distribution has access to the GPU (usually handled by Docker Desktop).
 
- Exiting and Re-entering the Container:
  - If you exit the container, it will stop. To start it again: `docker start mjpc_env`
  - To get a shell in the already running container: `docker exec -it mjpc_env /bin/bash`
 
- `MUJOCO_GL` Environment Variable: This variable is important for MuJoCo to know which rendering backend to use.
  - `egl`: For hardware-accelerated headless rendering (can also be used with X11 forwarding for a GUI window). Preferred for GPU.
  - `osmesa`: For CPU-based offscreen rendering
  - `glfw`: For creating a window directly using GLFW (requires X11 forwarding).


