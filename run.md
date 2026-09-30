# Sionna Research Kit — B210 OTA gNB + 5G Core

This setup runs the **OAI 5G Core + CUDA-enabled OAI gNB** on the DGX Spark, using a physical **USRP B210** for OTA experiments.

No RF simulator or OAI software UE is used.

## Normal Startup

### 1. Go to the Sionna Research Kit

```bash
cd ~/sionna-rk
```

### 2. Check that the B210 is detected

```bash
uhd_find_devices
```

Optional detailed check:

```bash
uhd_usrp_probe
```

### 3. Start the 5G Core Network

```bash
./scripts/start_system.sh 
```

Check that the core containers are running:

```bash
docker ps
```

### 4. Start the CUDA gNB with the B210

With X11 NR scope: 
```bash
docker run --rm -it   --gpus all   --network host   --privileged   -e DISPLAY=$DISPLAY   -v /tmp/.X11-unix:/tmp/.X11-unix:rw   -v /dev/bus/usb:/dev/bus/usb   -v ~/sionna-rk/nr-softmodem-dev:/opt/oai-gnb/bin/nr-softmodem:ro   -v ~/sionna-rk/ext/openairinterface5g/targets/PROJECTS/GENERIC-NR-5GC/CONF/fiu_n78.conf:/opt/oai-gnb/etc/gnb.conf:ro   --entrypoint /opt/oai-gnb/bin/nr-softmodem   oai-gnb-cuda:latest   -O /opt/oai-gnb/etc/gnb.conf   --sa   -E   -d   --continuous-tx   --gNBs.[0].min_rxtxtime 6   --RUs.[0].if_freq 500000000 --RUs.[0].att_rx 20 --RUs.[0].att_tx 20 
```
Without (recommended) X11 NR scope:
```bash
docker run --rm -it   --gpus all   --network host   --privileged   -v /dev/bus/usb:/dev/bus/usb   -v ~/sionna-rk/nr-softmodem-dev:/opt/oai-gnb/bin/nr-softmodem:ro   -v ~/sionna-rk/ext/openairinterface5g/targets/PROJECTS/GENERIC-NR-5GC/CONF/fiu_n78.conf:/opt/oai-gnb/etc/gnb.conf:ro   --entrypoint /opt/oai-gnb/bin/nr-softmodem   oai-gnb-cuda:latest   -O /opt/oai-gnb/etc/gnb.conf   --sa   -E     --continuous-tx   --gNBs.[0].min_rxtxtime 6   --RUs.[0].if_freq 500000000 --RUs.[0].att_rx 20 --RUs.[0].att_tx 20
```

### 5. Launch the UE

```
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --ue-fo-compensation -E -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/nrue.uicc.conf --usrp-args "type=b200,clock_source=external" --RUs.[0].if_freq 2300000000 --RUs.[0].clock_src external --RUs.[0].att_rx 5 --RUs.[0].att_tx 10 
```

### 6. Stop the gNB

Press:

```text
Ctrl+C
```

Because the container was launched with `--rm`, it is automatically removed when `nr-softmodem` exits.

### 7. Stop the Core Network

```bash
./scripts/stop_system.sh
```



**IPERF TESTS**

_DOWNLINK_

On the UE
```
iperf -s -u -i 1 -B 12.1.1.33
```
Then, on the gNB
```
sudo docker exec -it oai-ext-dn iperf -u -t 86400 -i 1 -fk -B 192.168.72.135 -b 20M -c 12.1.1.33
```

_UPLINK_

On the gNB
```
sudo docker exec -it oai-ext-dn iperf -s -u -i 1 -B 192.168.72.135
```
Then, on the UE
```
sudo ip route replace 192.168.72.135/32   dev oaitun_ue1   src 12.1.1.33
```
```
iperf -u -t 1000000 -i 1 -fk -b 5M -c 192.168.72.135 -B 12.1.1.33
```
