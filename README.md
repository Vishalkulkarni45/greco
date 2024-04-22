# Greco

> [!WARNING]  
> This is a research project and hasn't been audited. Use in production at your own risk.

Circuit for proving the correct encryption under BFV fully homomorphic encryption scheme. Note that this can be also generalized to any RLWE-based algorithms. Paper -> https://eprint.iacr.org/2024/594.

### Generate Parameters

To generate the parameters for the secret key proof of encryption circuit run the following command:

```bash
python3 scripts/circuit_sk.py -n 4096 -qis '[                                      
    27424203952895201, 
    27424203952895203
]' -t 65537
```

Where `-n` is the degree of the cyclotomic polynomial that defines the ring, `-qis` is the list of moduli qis such that qis[i] is the modulus of the i-th CRT basis of the modulus q of the ciphertext space, `-t` is the plaintext modulus. The value of `𝜎` for the gaussian distribution is set to 3.2 by default.

You can modify these parameters to fit your needs. Note that the python script used to generate the inputs is largely unoptimized and can take a while to run for parameters with large `n`, since the polynomial multiplication is not done using NTT.

As a result:
- A file `./src/data/sk_enc_{n}_{qis_len}x{qis_bitsize}_{t}.json` is generated including the input to the circuit that can be used for testing for those specific parameters. It includes a random secret key, a random plaintext message and the corresponding ciphertext encrypted under the secret key.
- A file `./src/data/sk_enc_{n}_{qis_len}x{qis_bitsize}_{t}_zeroes.json` is generated. In this file all the coefficients of the input polynomials are set to zero. This input is used at key generation time, when the actual inputs are not known.
- A file `./src/constants/sk_enc_constants_{n}_{qis_len}x{qis_bitsize}_{t}.rs` is generated including the generic constants for the circuit. Note that we separate them from the input because these should be known at compile time.

On top of that, the console will print an estimatation of the number of advice cells needed to compile the circuit in halo2 considering a single advice column and a lookup table of size 2^8. Spoiler: around 90% of the constraints are generated by the range checks on the polynomial coefficients.

### Circuit

```
cargo build
cargo test --release -- --nocapture
```

The halo2 circuit is based on a fork of `axiom-eth` that implements two minor changes:

- `RlcCircuit` and `RlcExecutor` are included into a utils mod such that they can be consumed outside of the crate 
- The `RlcCircuitInstructions` are modified to enable equality constraints on instance column in Second Phase

### Benches

To extract the benchmarks run: 

```
cargo test --release bench_sk_enc_full_prover --features bench -- --nocapture
```

By default, the benchmarks are run on the input data generated by the python script described above. Namely,`n = 4096` with 2 `qis` of 55 bits each and `t = 65537`. Other test vectors ara available by unzipping `src/data/data.zip`

To run the benchmarks on different parameters you have to:

1. Modify the constants being imported. For example `use crate::constants::sk_enc_constants_8192_4x55_65537::...`
2. Modify the `file_path` within `bench_sk_enc_full_prover()` function to point to the new input data file. For example `let file_path = "src/data/sk_enc_8192_4x55_65537.json";`

Given the same parameters, the benchmarks will run the prover and verifier for different values of `k`, where `2**k` is the maximum number of rows in the advice columns. After setting a specific `k`, halo2-lib will configure the circuit accordingly. With less rows at disposal, the configuration will contain more columns, and viceversa. 

More columns means more parallelism, which generally means faster proving times. Up to a point. Quoting [halo2-lib docs](https://docs.axiom.xyz/protocol/zero-knowledge-proofs/getting-started-with-halo2#feature-more-columns), there is a limit to parallelism, so generally the proving time vs. number of columns plot looks parabolic: as you increase number of columns (while keeping number of cells the same), the proving time will decrease up to a point before starting to increase again. 

### Results

Benchmarks run on M2 Macbook Pro with 12 cores and 32GB of RAM. 

The parameters have been chosen targeting 128-bit security level for different values of `n`. For more information on parameters choise, please check [Homomorphic Encryption Standard](https://homomorphicencryption.org/wp-content/uploads/2018/11/HomomorphicEncryptionStandardv1.1.pdf). 

Given the same parameters, the benchmarks will run the prover and verifier for different values of `k`, where `2**k` is the maximum number of rows in the advice columns. After setting a specific `k`, halo2-lib will configure the circuit accordingly. With less rows at disposal, the configuration will contain more columns, and viceversa. 

More columns means more parallelism, which generally means faster proving times. Up to a point. Quoting [halo2-lib docs](https://docs.axiom.xyz/protocol/zero-knowledge-proofs/getting-started-with-halo2#feature-more-columns), there is a limit to parallelism, so generally the proving time vs. number of columns plot looks parabolic: as you increase number of columns (while keeping number of cells the same), the proving time will decrease up to a point before starting to increase again. 

For certain parameters, low values of `k` might result in errors due to the lack of advice columns. That's why for certain parameters the benchmarks start at higher values of `k`.

#### `N = 1024`

```bash
python3 scripts/circuit_sk.py -n 1024 -qis '[                                      
    82638181
]' -t 65537
```

```
bfv params: "src/data/sk_enc_1024_1x27_65537"
+----+--------------------+--------------------+-----------------------+-------------------------+
| K  | VK Generation Time | PK Generation Time | Proof Generation Time | Proof Verification Time |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 12 | 459.975208ms       | 97.593417ms        | 776.78975ms           | 5.264583ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 13 | 376.945083ms       | 82.856458ms        | 685.505709ms          | 3.661792ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 14 | 327.682125ms       | 83.545375ms        | 771.287958ms          | 2.984834ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 15 | 353.973334ms       | 98.673167ms        | 927.688292ms          | 2.498542ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 16 | 562.989334ms       | 135.40525ms        | 1.479128167s          | 2.474292ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 17 | 771.174084ms       | 230.993791ms       | 2.5907405s            | 2.771833ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 18 | 1.448189792s       | 388.711083ms       | 4.966078959s          | 2.502875ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
```

#### `N = 2048`

```bash
python3 scripts/circuit_sk.py -n 2048 -qis '[                                      
    2869751532719597
]' -t 65537
```

```
bfv params: "src/data/sk_enc_2048_1x53_65537"
+----+--------------------+--------------------+-----------------------+-------------------------+
| K  | VK Generation Time | PK Generation Time | Proof Generation Time | Proof Verification Time |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 12 | 984.314167ms       | 209.913583ms       | 1.597872292s          | 8.236375ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 13 | 878.817375ms       | 193.876416ms       | 1.4118565s            | 5.399292ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 14 | 771.242375ms       | 186.790416ms       | 1.386890583s          | 3.735833ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 15 | 669.985041ms       | 190.327167ms       | 1.395613667s          | 3.198541ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 16 | 849.207834ms       | 219.047875ms       | 1.786423708s          | 2.649333ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 17 | 978.258416ms       | 288.281ms          | 2.821929417s          | 2.700958ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 18 | 1.477394209s       | 452.682833ms       | 5.152408292s          | 2.576083ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
```

#### `N = 4096`

```bash
python3 scripts/circuit_sk.py -n 4096 -qis '[                                      
    27424203952895201, 
    27424203952895203
]' -t 65537
```

```
bfv params: "src/data/sk_enc_4096_2x55_65537"
+----+--------------------+--------------------+-----------------------+-------------------------+
| K  | VK Generation Time | PK Generation Time | Proof Generation Time | Proof Verification Time |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 12 | 3.195995042s       | 759.2565ms         | 5.046092875s          | 20.572291ms             |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 13 | 2.791800458s       | 649.398458ms       | 4.3928835s            | 12.524709ms             |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 14 | 2.357667417s       | 631.544333ms       | 4.036068458s          | 7.709ms                 |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 15 | 2.1610035s         | 579.862292ms       | 3.474897667s          | 5.023917ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 16 | 1.986164917s       | 590.40925ms        | 3.76638225s           | 3.719708ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 17 | 1.684586375s       | 571.34275ms        | 3.785945209s          | 2.992958ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 18 | 2.234584458s       | 745.59725ms        | 6.14576425s           | 2.646542ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
```

#### `N = 8192`

```bash
python3 scripts/circuit_sk.py -n 8192 -qis '[                                      
    33676195539292805,
    33676195539292807, 
    33676195539292809, 
    33676195539292811
]' -t 65537
```

```
bfv params: "src/data/sk_enc_8192_4x55_65537"
+----+--------------------+--------------------+-----------------------+-------------------------+
| K  | VK Generation Time | PK Generation Time | Proof Generation Time | Proof Verification Time |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 12 | 12.015320708s      | 3.287664166s       | 17.841099166s         | 76.339042ms             |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 13 | 10.122673209s      | 2.693322459s       | 14.74348625s          | 37.897917ms             |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 14 | 8.418134917s       | 2.40066075s        | 12.849173459s         | 20.831916ms             |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 15 | 6.549789916s       | 2.019510917s       | 9.986622458s          | 11.373042ms             |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 16 | 5.803876875s       | 1.875661917s       | 9.325606542s          | 7.393ms                 |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 17 | 5.049139291s       | 1.819083667s       | 8.978745s             | 4.181208ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 18 | 4.78612025s        | 1.866191166s       | 10.295328167s         | 3.325291ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
```

#### `N = 16384`

```bash
python3 scripts/circuit_sk.py -n 16384 -qis '[                                      
    12905032264241935,
    12905032264241937, 
    12905032264241939, 
    12905032264241941, 
    12905032264241947, 
    12905032264241951, 
    12905032264241953, 
    12905032264241957
]' -t 65537
```

```
bfv params: "src/data/sk_enc_16384_8x54_65537"
+----+--------------------+--------------------+-----------------------+-------------------------+
| K  | VK Generation Time | PK Generation Time | Proof Generation Time | Proof Verification Time |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 13 | 42.00265175s       | 14.450244625s      | 56.991641875s         | 158.753791ms            |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 14 | 35.048573583s      | 11.7795115s        | 48.7445865s           | 71.56725ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 15 | 25.851191458s      | 9.141273625s       | 37.561299208s         | 32.032583ms             |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 16 | 22.46793575s       | 8.425404167s       | 34.270651s            | 17.28725ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 17 | 20.218826084s      | 8.164861959s       | 32.454668959s         | 10.2455ms               |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 18 | 17.590890541s      | 7.103390958s       | 29.432529166s         | 6.9695ms                |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 19 | 15.177152s         | 6.799758542s       | 30.783108166s         | 3.855708ms              |
+----+--------------------+--------------------+-----------------------+-------------------------+
```

#### `N = 32768`

```bash
python3 scripts/circuit_sk.py -n 32768 -qis '[                                      
    477501974462976263,
    477501974462976265,
    477501974462976267,
    477501974462976269,
    477501974462976271,
    477501974462976277,
    477501974462976283,
    477501974462976289,
    477501974462976293,
    477501974462976299,
    477501974462976301,
    477501974462976307,
    477501974462976311,
    477501974462976313,
    477501974462976317
]' -t 65537
```
```
bfv params: "src/data/sk_enc_32768_15x59_65537"
+----+--------------------+--------------------+-----------------------+-------------------------+
| K  | VK Generation Time | PK Generation Time | Proof Generation Time | Proof Verification Time |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 14 | 149.354869083s     | 65.239957417s      | 177.298666333s        | 337.225833ms            |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 15 | 115.596876125s     | 51.89580625s       | 154.238312375s        | 147.297625ms            |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 16 | 94.846151459s      | 40.290301042s      | 130.1495495s          | 61.721333ms             |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 17 | 77.865249s         | 34.1876465s        | 111.415501042s        | 26.904083ms             |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 18 | 64.948204541s      | 29.328586666s      | 102.15197475s         | 14.061625ms             |
+----+--------------------+--------------------+-----------------------+-------------------------+
| 19 | 62.038049125s      | 29.776692208s      | 104.779942208s        | 8.43725ms               |
+----+--------------------+--------------------+-----------------------+-------------------------+
```
