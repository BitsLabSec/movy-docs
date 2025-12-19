# Movy

![movy](./img/movy.jpg)

**Movy** is a Move testing framework that offers:

- Modular low-level building bricks for Move language. Specifically, the executor and tracer abstractions and layered database design borrowed from [revm](https://github.com/bluealloy/revm) that allow you to emulate and inspect an execution.
- Static analysis capabilities inherited from [MoveScan](https://dl.acm.org/doi/10.1145/3650212.3680391), the state-of-the-art static analyzer.
- Cutting-edge fuzzing reimplemented from scratch learned from [Belobog](https://github.com/abortfuzz/belobog) that supports both property testing and on-chain fuzzing, in the a flavor similar to [foundry](https://getfoundry.sh/forge/advanced-testing/overview) by writing invariants in Move language.
- And a lot of more...

This book serves as documents and tutorials for Movy.

> Please note that Movy is at __pre-alpha__ stage, which typically means it could contain bugs. However, rest assured that Movy **never** makes any real transactions to the live chain and thus your assets and private keys will always remain safe.

## Citation

Consider citing our works:

```bibtex
@misc{belobog,
      title={Belobog: Move Language Fuzzing Framework For Real-World Smart Contracts}, 
      author={Wanxu Xia and Ziqiao Kong and Zhengwei Li and Yi Lu and Pan Li and Liqun Yang and Yang Liu and Xiapu Luo and Shaohua Li},
      year={2025},
      eprint={2512.02918},
      archivePrefix={arXiv},
      primaryClass={cs.CR},
      url={https://arxiv.org/abs/2512.02918}, 
}

@inproceedings{movescan,
    author = {Song, Shuwei and Chen, Jiachi and Chen, Ting and Luo, Xiapu and Li, Teng and Yang, Wenwu and Wang, Leqing and Zhang, Weijie and Luo, Feng and He, Zheyuan and Lu, Yi and Li, Pan},
    title = {Empirical Study of Move Smart Contract Security: Introducing MoveScan for Enhanced Analysis},
    year = {2024},
    isbn = {9798400706127},
    publisher = {Association for Computing Machinery},
    address = {New York, NY, USA},
    url = {https://doi.org/10.1145/3650212.3680391},
    doi = {10.1145/3650212.3680391},
    abstract = {Move, a programming language for smart contracts, stands out for its focus on security. However, the practical security efficacy of Move contracts remains an open question. This work conducts the first comprehensive empirical study on the security of Move contracts. Our initial step involves collaborating with a security company to manually audit 652 contracts from 92 Move projects. This process reveals eight types of defects, with half previously unreported. These defects present potential security risks, cause functional flaws, mislead users, or waste computational resources. To further evaluate the prevalence of these defects in real-world Move contracts, we present MoveScan, an automated analysis framework that translates bytecode into an intermediate representation (IR), extracts essential meta-information, and detects all eight defect types. By leveraging MoveScan, we uncover 97,028 defects across all 37,302 deployed contracts in the Aptos and Sui blockchains, indicating a high prevalence of defects. Experimental results demonstrate that the precision of MoveScan reaches 98.85\%, with an average project analysis time of merely 5.45 milliseconds. This surpasses previous state-of-the-art tools MoveLint, which exhibits an accuracy of 87.50\% with an average project analysis time of 71.72 milliseconds, and Move Prover, which has a recall rate of 6.02\% and requires manual intervention. Our research also yields new observations and insights that aid in developing more secure Move contracts.},
    booktitle = {Proceedings of the 33rd ACM SIGSOFT International Symposium on Software Testing and Analysis},
    pages = {1682–1694},
    numpages = {13},
    keywords = {Defect, Move language, Program analysis, Smart contract},
    location = {Vienna, Austria},
    series = {ISSTA 2024}
    }
```