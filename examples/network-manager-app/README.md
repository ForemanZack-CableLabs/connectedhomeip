# network-manager-app

This is a reference application that implements the Network Infrastructure
Manager device type.

## PQC device attestation (Linux)

The application accepts `--dac_provider <json-file>` through the shared Linux
example options. Loading a complete PQC PAI/DAC pair enables the Operational
Credentials `PQCDA` feature and `PQCDeviceAttestationProfile` automatically.

The development fixtures in
[`credentials/development/attestation/pqc`](../../credentials/development/attestation/pqc/README.md)
provide these chain sets for VID `0xFFF1`, PID `0x8000`:

| Fixture | PAA and PAI algorithms |
| --- | --- |
| `TestCredentials-FFF1-8000-PQC-Legacy.json` | ECDSA, ML-DSA-44, ML-DSA-65 |
| `TestCredentials-FFF1-8000-ML-DSA-44-Legacy.json` | ECDSA, ML-DSA-44 |
| `TestCredentials-FFF1-8000-ML-DSA-65-Legacy.json` | ECDSA, ML-DSA-65 |

Every chain uses the same P-256 DAC key for Device Attestation signatures.
The combined fixture lets the test harness select ML-DSA-65; the ML-DSA-44
fixture exercises ML-DSA-44 explicitly. These credentials are for development
and testing only.

Build and launch from the repository root:

```bash
scripts/run_in_build_env.sh './scripts/build/build_examples.py --target linux-x64-network-manager-no-ble-clang build'
./out/linux-x64-network-manager-no-ble-clang/matter-network-manager-app \
  --dac_provider credentials/development/attestation/pqc/TestCredentials-FFF1-8000-PQC-Legacy.json \
  --vendor-id 65521 --product-id 32768 \
  --discriminator 1234 --passcode 20202021 \
  --KVS /tmp/nim-pqc.kvs
```

Use a fresh KVS path when commissioning a new test instance. The app serves
provisioned certificates as bytes, so its crypto backend does not need
ML-DSA signing capability to serve these chains.

## Run OPCREDS-3.9 and DA-1.10

Build the Python controller using `scripts/build_python.sh -i out/venv` as
described in the [Python testing guide](../../docs/testing/python.md).
The bindings must include the PQC Operational Credentials definitions.
The Python environment must provide
`cryptography.hazmat.primitives.asymmetric.mldsa` for DA-1.10 to verify the
ML-DSA signatures independently of the SDK crypto backend.

Run the unchanged certification scripts against the application above:

```bash
source out/venv/bin/activate
python3 src/python_testing/TC_OPCREDS_3_9.py \
  --storage-path /tmp/nim-pqc-controller.json \
  --commissioning-method on-network --discriminator 1234 --passcode 20202021 \
  --paa-trust-store-path credentials/development/attestation/pqc/th-paa-root-certs
python3 src/python_testing/TC_DA_1_10.py \
  --storage-path /tmp/nim-pqc-controller.json \
  --paa-trust-store-path credentials/development/attestation/pqc/th-paa-root-certs
```

The second command reuses the commissioned fabric. DA-1.10 needs the supplied
trust store because it validates both the selected PQC chain and the legacy
ECDSA chain. For a remote test harness, copy `th-paa-root-certs/` to that host
and use its local path; the credential JSON stays on the DUT.
