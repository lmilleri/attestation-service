# Attestation Service

Attestation Service (AS for short) is a general function set that can verify TEE evidence.
With Confidential Containers, the attestation service must run in an secure environment, outside of the guest node.

With remote attestation, Attesters (e.g. the [Attestation Agent](https://github.com/confidential-containers/attestation-agent)) running on the guest node will request a resource (e.g. a container image decryption key) from the [Key Broker Service (KBS)](https://github.com/confidential-containers/kbs)).
The KBS receives the attestation evidence from the client in TEE and forwards it to the Attestation Service (AS). The AS role is to verify the attestation evidence and provide Attestation Results token back to the KBS. Verifying the evidence is a two steps process:

1. Verify the evidence signature, and assess that it's been signed with a trusted key of TEE.
2. Verify that the TCB described by that evidence (including hardware status and software measurements) meets the guest owner expectations.

Those two steps are accomplished by respectively one of the [Verifier Drivers](#verifier-drivers) and the AS [Policy Engine](#policy-engine). The policy engine can be customized with different policy configurations.

The AS can be built as a library (i.e. a Rust crate) by other confidential computing resources providers, like for example the KBS.
It can also run as a standalone binary, in which case its API is exposed through a set of [gRPC](https://grpc.io/) endpoints.

# Components

## Library

The AS can be built and imported as a Rust crate into any project providing attestation services.

As the AS API is not yet fully stable, the AS crate needs to be imported from GitHub directly:

```toml
attestation-service = { git = "https://github.com/confidential-containers/attestation-service", branch = "main" }
```

## Server

This project provides the Attestation Service binary program that can be run as an independent server:

- [`grpc-as`](bin/grpc-as/): Provide AS APIs based on gRPC protocol.

# Usage

Build and install AS components:

```shell
git clone https://github.com/confidential-containers/attestation-service
cd attestation-service
make && make install
```

`grpc-as` will be installed into `/usr/local/bin`.

# Architecture

The main architecture of the Attestation Service is shown in the figure below:
```
                                      ┌───────────────────────┐
┌───────────────────────┐ Evidence    │  Attestation Service  │
│                       ├────────────►│                       │
│ Verification Demander │             │ ┌────────┐ ┌──────────┴───────┐
│    (Such as KBS)      │             │ │ Policy │ │ Reference Value  │◄───Reference Value
│                       │◄────────────┤ │ Engine │ │ Provider Service │
└───────────────────────┘ Attestation │ └────────┘ └──────────┬───────┘
                        Results Token │                       │
                                      │ ┌───────────────────┐ │
                                      │ │  Verifier Drivers │ │
                                      │ └───────────────────┘ │
                                      │                       │
                                      └───────────────────────┘
```

The Reference Value Provider Service supports different deploy mode,
please refer to [the doc](./docs/rvps.md#run-mode) for more details.

### Evidence format:

The attestation evidence is included in a [KBS defined Attestation payload](https://github.com/confidential-containers/kbs/blob/main/docs/kbs_attestation_protocol.md#attestation):

```json
{
    "tee-pubkey": $pubkey,
    "tee-evidence": {}
}
```

- `tee-pubkey`: A JWK-formatted public key, generated by the client running in the HW-TEE.
For more details on the `tee-pubkey` format, see the [KBS protocol](https://github.com/confidential-containers/kbs/blob/main/docs/kbs_attestation_protocol.md#key-format).

- `tee-evidence`: The attestation evidence generated by the HW-TEE platform software and hardware in the AA's execution environment.
The tee-evidence formats depend on the TEE and are typically defined by the each TEE verifier driver of AS.

**Note**: Verification Demander needs to specify the TEE type and pass `nonce` to Attestation-Service together with Evidence,
Hash of `nonce` and `tee-pubkey` should be embedded in report/quote in `tee-evidence`, so they can be signed by HW-TEE.
This mechanism ensures the freshness of Evidence and the authenticity of `tee-pubkey`.

### Attestation Results Token:

If the verification of TEE evidence is successful, AS will return an Attestation Results Token.
Otherwise, AS will return an Error which contain verifier output or policy engine output.

Attestation results token is a [JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519) which contains the parsed evidence claims such as TCB status.

Claims format of the attestation results token is:

```json
{
    "iss": $issuer_name,
    "jwk": $public_key,
    "exp": $expire_timestamp,
    "nbf": $notbefore_timestamp,
    "tee-pubkey": $pubkey,
    "tcb-status": $parsed_evidence,
    "evaluation-report": $report
}
```

* `iss`: Token issuer name, default is `CoCo-Attestation-Service`.
* `jwk`: Public key to verify token signature. Must be in format of [JSON Web Key](https://datatracker.ietf.org/doc/html/rfc7517).
* `exp`: Token expire time in Unix timestamp format.
* `nbf`: Token effective time in Unix timestamp format.
* `tee-pubkey`: A JWK-formatted public key, generated by the client running in the HW-TEE.
For more details on the `tee-pubkey` format, see the [KBS protocol](https://github.com/confidential-containers/kbs/blob/main/docs/kbs_attestation_protocol.md#key-format).
* `tcb_status`: Contains HW-TEE informations and software measurements of AA's execution environment.
* `evaluation-report` : The output of the policy engine, it is AS policy's evaluation opinion on TEE evidence.

## Verifier Drivers

A verifier driver parse the HW-TEE specific `tee-evidence` data from the received attestation evidence, and performs the following tasks:

1. Verify HW-TEE hardware signature of the TEE quote/report in `tee-evidence`.

2. Resolve `tee-evidence`, and organize the TCB status into JSON claims to return.

Supported Verifier Drivers:

- `sample`: A dummy TEE verifier driver which is used to test/demo the AS's functionalities.
- `tdx`: Verifier Driver for Intel Trust Domain Extention (Intel TDX).
- `amd-sev-snp`: TODO.

## Policy Engine

The AS supports modular policy engine, which can be specified through the AS configuration. The currently supported policy engines are:

### [Open Policy Agent (OPA)](https://www.openpolicyagent.org/docs/latest/)

OPA is a very flexible and powerful policy engine, AS allows users to define and upload their own policy when performing evidence verification.

**Note**: Please refer to the [Policy Language](https://www.openpolicyagent.org/docs/latest/policy-language/) documentation for more information about the `.rego`.

If the user does not need to customize his own policy, AS will use the [default policy](src/policy_engine/opa/default_policy.rego).

## Reference Value Provider

[Reference Value Provider Service](docs/rvps.md) (RVPS for short) is a module integrated in the AS to verify,
store and provide reference values. RVPS receives and verifies the provenance input from the software supply chain,
stores the measurement values, and generates reference value claims for the AS according to the evidence content when the AS verifies the evidence.