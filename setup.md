# DGX Spark + Sionna Research Kit + OAI — Complete Installation Guide

This guide documents the complete procedure for installing and running the **NVIDIA Sionna Research Kit (Sionna-RK) with OpenAirInterface 5G on NVIDIA DGX Spark**, starting from a fresh DGX Spark.

It also documents the `.gitmodules` patch issue encountered with Sionna-RK v1.3.2 and how to recover from it.

---

# 1. Verify the DGX Spark Architecture

Open a terminal and run:

```bash id="qvwoxd"
uname -m
dpkg --print-architecture
```

Expected:

```text id="tdf7nl"
aarch64
arm64
```

**Important:** DGX Spark is ARM64, not AMD64/x86-64.

When downloading `.deb` packages, always choose:

```text id="1ds8qm"
*_arm64.deb
```

not:

```text id="2gc5xe"
*_amd64.deb
```

For example, for VS Code:

```text id="e9fjpp"
code_xxx_arm64.deb     # CORRECT
code_xxx_amd64.deb     # WRONG
```

Do not attempt to solve an architecture mismatch by installing `:amd64` dependencies.

---

# 2. Make Sure Git and Docker Are Available

Check:

```bash id="iwvdn"
git --version
docker --version
```

Also verify Docker works:

```bash id="w2d7j7"
docker ps
```

If Docker responds normally, continue.

---

# 3. Clone the NVIDIA Sionna Research Kit

Go to your home directory:

```bash id="m9w22c"
cd ~
```

Clone Sionna-RK:

```bash id="7y6m2v"
git clone https://github.com/NVlabs/sionna-rk.git
```

Enter the repository:

```bash id="xdj5dx"
cd ~/sionna-rk
```

Check the version:

```bash id="3wev0a"
git describe --tags --always
```

For the installation documented here, the known working version is:

```text id="u3nvrn"
v1.3.2
```

with commit:

```text id="vhidw7"
0abdce3ff7f5840d64b783632eb419ea6bf54ae5
```

If you specifically want to reproduce this known-working setup:

```bash id="1dz1do"
git checkout v1.3.2
```

---

# 4. Initial Sionna-RK / OAI Setup

From:

```bash id="suw5zv"
cd ~/sionna-rk
```

run:

```bash id="3dpnpl"
make sionna-rk
```

This invokes the Sionna-RK OAI setup process, including:

```text id="9qbspl"
./scripts/quickstart-oai.sh
```

and creates the OAI tree under:

```text id="ozpxdi"
~/sionna-rk/ext/openairinterface5g
```

For Sionna-RK v1.3.2, the setup used OAI:

```text id="p1cg7n"
2025.w34
```

at commit:

```text id="5j4r1s"
207aac94d13e714abac8062ebf869805f2acd6dc
```

---

# 5. If the OAI Directory Already Exists

If you get:

```text id="q8rhnj"
Destination directory .../ext/openairinterface5g already exists.
Use the --clean option or remove it before proceeding.
```

run:

```bash id="ihjs4a"
cd ~/sionna-rk

./scripts/quickstart-oai.sh --clean
```

**Warning:** `--clean` deletes and recreates:

```text id="01n0c7"
~/sionna-rk/ext/openairinterface5g
```

Do not do this if you have custom OAI changes you need to preserve.

---

# 6. Known v1.3.2 `.gitmodules` Patch Problem

During setup on the DGX Spark, we encountered:

```text id="36otdy"
Applying SRK patches...

error: patch failed: .gitmodules:1
error: .gitmodules: patch does not apply
```

This is important.

Because the Sionna-RK patch does not finish applying, the DGX Spark-specific Dockerfiles are not created.

The subsequent build therefore fails with errors such as:

```text id="2yt6ys"
Dockerfile.base.ubuntu.cuda: no such file or directory
Dockerfile.build.ubuntu.cuda: no such file or directory
Dockerfile.gNB.ubuntu.cuda: no such file or directory
Dockerfile.nrUE.ubuntu.cuda: no such file or directory
Dockerfile.flexric.ubuntu: no such file or directory
```

---

# 7. Verify That `.gitmodules` Is the Only Patch Failure

Go into the newly cloned OAI repository:

```bash id="dx87xg"
cd ~/sionna-rk/ext/openairinterface5g
```

Run:

```bash id="w9hxhx"
git apply --check ~/sionna-rk/patches/openairinterface5g.patch
```

For our setup, the result was only:

```text id="iih2bn"
error: patch failed: .gitmodules:1
error: .gitmodules: patch does not apply
```

If **other files also fail**, do not blindly continue with the workaround below.

---

# 8. Apply the Rest of the Sionna-RK Patch

If `.gitmodules` is the **only** failure:

```bash id="blve3h"
cd ~/sionna-rk/ext/openairinterface5g

git apply --reject ~/sionna-rk/patches/openairinterface5g.patch
```

You should see:

```text id="9gtamx"
Applying patch .gitmodules with 1 reject...
Rejected hunk #1.
```

followed by successful application of the important patches:

```text id="ssx5v9"
Applied patch docker/Dockerfile.base.ubuntu.cuda cleanly.
Applied patch docker/Dockerfile.build.ubuntu.cuda cleanly.
Applied patch docker/Dockerfile.flexric.ubuntu cleanly.
Applied patch docker/Dockerfile.gNB.ubuntu.cuda cleanly.
Applied patch docker/Dockerfile.nrUE.ubuntu.cuda cleanly.
```

along with the other OAI patches.

Warnings about trailing whitespace are harmless.

You may also see:

```text id="uhopkf"
warning: unable to rmdir 'openair2/E2AP/flexric':
Directory not empty
```

This can occur because the FlexRIC submodule has already been initialized.

---

# 9. Verify the DGX Spark Dockerfiles

Run:

```bash id="9flv8w"
cd ~/sionna-rk/ext/openairinterface5g

ls docker | grep -E 'cuda|flexric'
```

You should now have:

```text id="gakg1i"
Dockerfile.base.ubuntu.cuda
Dockerfile.build.ubuntu.cuda
Dockerfile.flexric.ubuntu
Dockerfile.gNB.ubuntu.cuda
Dockerfile.nrUE.ubuntu.cuda
```

Remove the rejected patch artifact:

```bash id="ghgj5n"
rm -f .gitmodules.rej
```

---

# 10. Build FlexRIC

From the OAI directory:

```bash id="9k0v4w"
cd ~/sionna-rk/ext/openairinterface5g
```

build:

```bash id="4azx01"
docker build \
  --build-arg DOCKER_CUSTOM_IMAGE_TAG=latest \
  --target oai-flexric-fixed \
  --tag oai-flexric:latest \
  --file docker/Dockerfile.flexric.ubuntu \
  .
```

Verify:

```bash id="vkw4d3"
docker images | grep flexric
```

You should see:

```text id="vscoy1"
oai-flexric    latest
```

---

# 11. Build the DGX Spark CUDA Base Image

```bash id="zjfq7u"
cd ~/sionna-rk/ext/openairinterface5g

docker build \
  --progress plain \
  --build-arg DOCKER_CUSTOM_IMAGE_TAG=latest \
  --target ran-base-cuda \
  --tag ran-base-cuda:latest \
  --file docker/Dockerfile.base.ubuntu.cuda \
  .
```

---

# 12. Build the CUDA Build Image

The build context for this image is the **Sionna-RK root**, not the OAI directory:

```bash id="ol0y20"
docker build \
  --progress plain \
  --build-arg DOCKER_CUSTOM_IMAGE_TAG=latest \
  --target ran-build-cuda \
  --tag ran-build-cuda:latest \
  --file ~/sionna-rk/ext/openairinterface5g/docker/Dockerfile.build.ubuntu.cuda \
  ~/sionna-rk
```

---

# 13. Build the OAI CUDA gNB

Follow this tutorial to compile the actual gNB: https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_nrUE.md?ref_type=heads

In particular, 

```
# Install OAI dependencies
cd ~/openairinterface5g/cmake_targets
./build_oai -I

# nrscope dependencies
sudo apt install -y libforms-dev libforms-bin

# Build OAI gNB
cd ~/openairinterface5g/cmake_targets
./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
```


```bash id="avlmxd"
cd ~/sionna-rk/ext/openairinterface5g

docker build \
  --progress plain \
  --build-arg DOCKER_CUSTOM_IMAGE_TAG=latest \
  --target oai-gnb-cuda \
  --tag oai-gnb-cuda:latest \
  --file docker/Dockerfile.gNB.ubuntu.cuda \
  .
```

Verify:

```bash id="asgxzi"
docker images | grep oai-gnb-cuda
```

---

# 14. Build the OAI CUDA UE

For RF simulator experiments, also build:

```bash id="bj2gya"
cd ~/sionna-rk/ext/openairinterface5g

docker build \
  --progress plain \
  --build-arg DOCKER_CUSTOM_IMAGE_TAG=latest \
  --target oai-nr-ue-cuda \
  --tag oai-nr-ue-cuda:latest \
  --file docker/Dockerfile.nrUE.ubuntu.cuda \
  .
```

---

# 15. Verify All Required Local Images

Run:

```bash id="yyv7aa"
docker images | grep -E 'ran-base-cuda|ran-build-cuda|oai-gnb-cuda|oai-nr-ue-cuda|oai-flexric'
```

You want to see:

```text id="o8yz5x"
ran-base-cuda:latest
ran-build-cuda:latest
oai-gnb-cuda:latest
oai-nr-ue-cuda:latest
oai-flexric:latest
```

These are important because Sionna-RK expects these images to exist **locally**.

---

# 16. Configure Sionna-RK

The runtime configuration is under:

```text id="m1fl22"
~/sionna-rk/config/
```

For RF simulator operation, the configuration used here is:

```text id="m5vkvv"
~/sionna-rk/config/rfsim/
```

with:

```text id="1zh7kh"
~/sionna-rk/config/rfsim/.env
```

When starting the system, Sionna-RK should report:

```text id="y1ebvz"
Using config: rfsim
(env: /home/marco/sionna-rk/config/rfsim/.env)
```

---

# 17. Start the Complete 5G System

Now run:

```bash id="otmbsa"
cd ~/sionna-rk

./scripts/start_system.sh rfsim
```

The first stage starts the OAI 5G Core.

A healthy startup looks like:

```text id="ws46ne"
Starting 5G Core network

oai-mysql is ready!
oai-amf is ready!
oai-smf is ready!
oai-upf is ready!
oai-ext-dn is ready!

All services are up and healthy!
```

Then Sionna-RK starts the gNB:

```text id="4dn10f"
Starting gNB
```

and subsequently the rest of the RF simulator setup.

---

# 18. Understanding `pull access denied`

If you see:

```text id="iq9q71"
pull access denied for oai-flexric
```

or:

```text id="epg54s"
pull access denied for oai-gnb-cuda
```

do **NOT** immediately run:

```bash id="m2f1d9"
docker login
```

The likely problem is that the expected **local Docker image does not exist**.

For FlexRIC:

```bash id="dfhnza"
docker images | grep flexric
```

For the gNB:

```bash id="s40jyw"
docker images | grep oai-gnb-cuda
```

For the UE:

```bash id="zdsx3g"
docker images | grep oai-nr-ue-cuda
```

If the expected image is missing, build it locally using the instructions above.

---

# 19. Normal Daily Startup

Once the complete installation has been performed successfully, **you do NOT need to rebuild anything when you reboot the DGX Spark**.

Turn on the Spark, open a terminal, and simply run:

```bash id="43t0eo"
cd ~/sionna-rk

./scripts/start_system.sh rfsim
```

That's the normal workflow.

Optionally, before starting:

```bash id="18nk7z"
docker images | grep -E 'oai-flexric|oai-gnb-cuda|oai-nr-ue-cuda'
```

If those images exist, you're ready.

---

# 20. Quick From-Scratch Installation Summary

Starting from a fresh DGX Spark:

```bash id="ynhz1d"
# 1. Check architecture
uname -m
# Expected: aarch64

# 2. Clone Sionna-RK
cd ~
git clone https://github.com/NVlabs/sionna-rk.git
cd sionna-rk

# 3. Use known-working release
git checkout v1.3.2

# 4. Initial setup
make sionna-rk
```

If the setup fails because of `.gitmodules`:

```bash id="p7imvf"
cd ~/sionna-rk/ext/openairinterface5g

git apply --check ~/sionna-rk/patches/openairinterface5g.patch

git apply --reject ~/sionna-rk/patches/openairinterface5g.patch

rm -f .gitmodules.rej
```

Then build the images:

```bash id="9q13hu"
# FlexRIC
cd ~/sionna-rk/ext/openairinterface5g

docker build \
  --build-arg DOCKER_CUSTOM_IMAGE_TAG=latest \
  --target oai-flexric-fixed \
  --tag oai-flexric:latest \
  --file docker/Dockerfile.flexric.ubuntu \
  .

# CUDA base
docker build \
  --progress plain \
  --build-arg DOCKER_CUSTOM_IMAGE_TAG=latest \
  --target ran-base-cuda \
  --tag ran-base-cuda:latest \
  --file docker/Dockerfile.base.ubuntu.cuda \
  .

# CUDA build image
docker build \
  --progress plain \
  --build-arg DOCKER_CUSTOM_IMAGE_TAG=latest \
  --target ran-build-cuda \
  --tag ran-build-cuda:latest \
  --file ~/sionna-rk/ext/openairinterface5g/docker/Dockerfile.build.ubuntu.cuda \
  ~/sionna-rk

# CUDA gNB
cd ~/sionna-rk/ext/openairinterface5g

docker build \
  --progress plain \
  --build-arg DOCKER_CUSTOM_IMAGE_TAG=latest \
  --target oai-gnb-cuda \
  --tag oai-gnb-cuda:latest \
  --file docker/Dockerfile.gNB.ubuntu.cuda \
  .

# CUDA UE
docker build \
  --progress plain \
  --build-arg DOCKER_CUSTOM_IMAGE_TAG=latest \
  --target oai-nr-ue-cuda \
  --tag oai-nr-ue-cuda:latest \
  --file docker/Dockerfile.nrUE.ubuntu.cuda \
  .
```

Verify:

```bash id="f7s9mq"
docker images | grep -E 'ran-base-cuda|ran-build-cuda|oai-gnb-cuda|oai-nr-ue-cuda|oai-flexric'
```

Finally:

```bash id="1uyum7"
cd ~/sionna-rk

./scripts/start_system.sh rfsim
```

---

# Golden Rules

1. **DGX Spark is ARM64.** Never accidentally install AMD64 packages.

2. `oai-flexric`, `oai-gnb-cuda`, and `oai-nr-ue-cuda` are expected to be **local Docker images**.

3. `pull access denied` for one of these images usually means **the local image is missing**, not that Docker authentication is required.

4. If the Dockerfiles themselves are missing, check whether the Sionna-RK OAI patch failed.

5. With the Sionna-RK v1.3.2 setup documented here, `.gitmodules` was the only rejected patch hunk. We successfully applied the remaining patch with:

```bash id="oq05kj"
git apply --reject ~/sionna-rk/patches/openairinterface5g.patch
```

6. Once the installation works, **do not rebuild everything on every boot**.

Normal operation is simply:

```bash id="8yhdhs"
cd ~/sionna-rk
./scripts/start_system.sh rfsim
```
