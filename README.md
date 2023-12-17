# Combined Attestation Project for Key Broker Service (KBS)

## Overview
This project demonstrates the integration of H100 attestation via nvTrust with CPU evidence, verified through the [Key Broker Service (KBS)](https://github.com/confidential-containers/kbs) as part of the Confidential Containers project. The aim is to provide a secure and scalable attestation method in a containerized environment.

## Demo
[Demo Video](https://www.youtube.com/watch?v=PUM6HVjNAm8)

## Prerequisites
- Ubuntu 22.04
- This project has been tested exclusively on Ubuntu 22.04.

## Installation and Configuration

### Set Up the Environment
Ensure your system is prepared with the necessary tools and libraries:
```bash
# Add Intel SGX repository and install dependencies
curl -L "https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key" | sudo apt-key add -
echo "deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu jammy main" \
  | sudo tee /etc/apt/sources.list.d/intel-sgx.list
sudo apt-get update
sudo apt-get install -y build-essential clang libsgx-dcap-quote-verify-dev \
  libsgx-dcap-quote-verify libtdx-attest-dev libtdx-attest libtss2-dev \
  openssl pkg-config protobuf-compiler

# Install Rust and Go
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
sudo apt-get install libssl-dev
wget https://go.dev/dl/go1.21.3.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.3.linux-amd64.tar.gz
echo "export PATH=$PATH:/usr/local/go/bin" >> ~/.bashrc
source ~/.bashrc
sudo apt-get install libpython3.11-dev
```

### Clone and Set Up Repositories
Clone and prepare the necessary repositories:
```bash
# KBS repository
git clone https://github.com/jingyao-zhang/kbs.git
cd kbs
git checkout cb-attest
source auto-build.sh

# nvTrust repository
cd ..
git clone https://github.com/jingyao-zhang/nvtrust.git
cd nvtrust
git checkout cb-attest

# Install Miniconda and set up the environment for nvTrust
cd ..
git clone https://github.com/jingyao-zhang/env-setting.git
cd env-setting
source conda-install.sh
conda create -n nvtrust python=3.10
conda activate nvtrust
pip install -r requirements.txt

# Install nvTrust in the Conda environment
cd ../nvtrust
cd guest_tools/gpu_verifiers/local_gpu_verifier/
pip install .
cd ../../attestation_sdk/
pip install .
cd tests
python LocalGPUTest.py
```

## Testing KBS
Follow these steps to test the KBS setup:
```bash
cd ../../../../kbs
# Generate keys
openssl genpkey -algorithm ed25519 > config/private.key
openssl pkey -in config/private.key -pubout -out config/public.pub

# Start KBS services
sudo RUST_LOG=debug issuer-kbs --config-file issue-kbs.toml
sudo resource-kbs --config-file resource-kbs.toml

# Test KBS client
cat > test/dummy_data << EOF
1234567890abcde
EOF

kbs-client --url http://127.0.0.1:50002 config --auth-private-key config/private.key set-resource --resource-file test/dummy_data --path default/test/dummy

openssl genrsa -traditional -out test/tee_key.pem 2048
openssl rsa -in test/tee_key.pem  -pubout -out test/tee_pubkey.pem

sudo RUST_LOG=debug AA_SAMPLE_ATTESTER_TEST=1 RUST_BACKTRACE=full kbs-client --url http://127.0.0.1:50001 attest --tee-key-file test/tee_key.pem > test/attestation_token

kbs-client --url http://127.0.0.1:50002 get-resource --attestation-token test/attestation_token --tee-key-file test/tee_key.pem --path default/test/dummy
```

## Contributing
We welcome contributions to enhance the Combined Attestation project.

## License
MIT License
