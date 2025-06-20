# zk-verify

✅ 1. CÀI ĐẶT MÔI TRƯỜNG
sudo apt update && sudo apt install -y curl git jq
npm install -g snarkjs circom

snarkjs --version
circom --version

✅ 2. BIÊN DỊCH CIRCUIT
mkdir zkverify && cd zkverify
mkdir -p circuits
nano circuits/multiplier.circom

template Multiplier() {
    signal input a;
    signal input b;
    signal input c;

    c <== a * b;
}

component main = Multiplier();

circom circuits/multiplier.circom --r1cs --wasm --sym -o circuits/

mv multiplier.* circuits/

✅ 3. TẠO PTAU & ZKEY
mkdir -p keys
# Tạo powers of tau ceremony
snarkjs powersoftau contribute keys/pot12_0000.ptau keys/pot12_final.ptau --name="YOUR-NAME" -v
>>Enter a random text. (Entropy): toanbm-2025-proof-challenge

snarkjs powersoftau prepare phase2 keys/pot12_final.ptau keys/pot12_final_prepared.ptau
snarkjs groth16 setup circuits/multiplier.r1cs keys/pot12_final_prepared.ptau keys/multiplier_0000.zkey
# Tạo zkey
snarkjs groth16 setup circuits/multiplier.r1cs keys/pot12_final.ptau keys/multiplier_0000.zkey
snarkjs zkey contribute keys/multiplier_0000.zkey keys/multiplier_final.zkey --name="YOUR-NAME" -v
>>Enter a random text. (Entropy): zkverify-challenge-2025

# Tạo verification key
snarkjs zkey export verificationkey keys/multiplier_final.zkey keys/verification_key.json

✅ 4. TẠO INPUT & WITNESS




# Tạo input đơn giản
mkdir -p input
echo '{ "a": 3, "b": 11, "c": 33 }' > input/input.json

# Tạo witness
mkdir -p witness
snarkjs wtns calculate circuits/multiplier.wasm input/input.json witness/witness.wtns

✅ 5. TẠO PROOF & PUBLIC

mkdir -p proofs
snarkjs groth16 prove keys/multiplier_final.zkey witness/witness.wtns proofs/proof.json proofs/public.json

✅ 6. TẠO FILE PAYLOAD

mkdir -p scripts
nano scripts/generatePayload.js
const fs = require("fs");

// Read files
const proof = JSON.parse(fs.readFileSync("proofs/proof.json", "utf8"));
const publicSignals = JSON.parse(fs.readFileSync("proofs/public.json", "utf8"));
const vk = JSON.parse(fs.readFileSync("keys/verification_key.json", "utf8"));

// Final payload format
const payload = {
  proofType: "groth16",
  vkRegistered: false,
  proofOptions: {
    library: "snarkjs",
    curve: "bn128"
  },
  proofData: {
    proof: {
      pi_a: proof.pi_a.map(String),
      pi_b: proof.pi_b.map(pair => pair.map(String)),
      pi_c: proof.pi_c.map(String)
    },
    publicSignals: publicSignals.map(String),
    vk
  }
};

fs.writeFileSync("payload.json", JSON.stringify(payload, null, 2));
console.log("✅ Correct payload.json created");


node scripts/generatePayload.js

✅ 7. GỬI LÊN RELAYER API

curl -X POST https://relayer-api.horizenlabs.io/api/v1/submit-proof/598f259f5f5d7476622ae52677395932fa98901f \
  -H "Content-Type: application/json" \
  -d @payload.json






