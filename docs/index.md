---
title: GPU移行
description: GPU移行に関するポータルサイト
---

<details open markdown="block">
<summary>目次</summary>
- TOC
{:toc}
</details>

# 背景

東京大学情報基盤センターと筑波大学計算科学研究センターが共同で運営する最先端共同HPC基盤施設（JCAHPC）のOakforest-PACSシステム（OFP）は2022年3月末で無事運用を終了いたしました。JCAHPCでは[後継機種の導入を決定](https://www.jcahpc.jp/pr/news-20240219.html)し、[次期スーパーコンピュータシステムの名称を “Miyabi”（みやび）に決定](https://www.jcahpc.jp/pr/news-20240401.html)しました。Miyabiシステムは、米国NVIDIA社による超高速CPU-GPU専用リンクNVLink-C2Cで接続したGH200 Grace-Hopper Superchipを搭載した計算ノード1,120ノード(Miyabi-G)と、米国Intel社によるXeon Max 9480を2基搭載した計算ノード190ノード(Miyabi-C)をInfiniBand NDR200で結合した、倍精度演算性能80.1 PFLOPSを有する超並列クラスタ型スーパーコンピュータであり、GH200を搭載した国内初の汎用大規模システムです。2025年1月より運用を開始いたしました。

本サイトでは、Miyabiの利用開始に向け、皆様の既存コードをご自身でGPUへ移植するために、役立つ情報を集約しています。

# GPU移行に関する情報

## 移行の概要

既存のCPUコードをNVIDIA GPUへ移植して高い性能を得るために大事なことは、

- ループなどの計算の重い部分をCPUからGPUへ移植する
- 配列などのデータを極力GPU上のメモリだけに保持する（CPUとGPU間のデータ通信を極力減らす）

です。これに対して、どこまで性能を追求するか、開発コストをかけられるか、ソースコードのメンテナンス性を高めるかで、選択すべき移植方法が変わってきます。

既存のCPUコードをNVIDIA GPUへ移植するためには、いくつかの方法があり、代表的な方法は次のとおりです。
1番から3番に行くにつれて、達成できる性能は高くなる傾向にありますが、反対に開発効率が低下していきます。
OpenMP + MPIで並列化済みであれば、一般的には OpenACC + MPI へ移行することがおすすめです。

1. GPU対応ライブラリを用いる
2. コンパイラの支援を利用してGPU移植する
   - 既存のCPUコードに指示文（ディレクティブ）を挿入する
     - OpenACCやOpenMPのtarget指示文といった選択肢があります
   - 標準言語の並列処理でGPUを利用する
3. GPU専用言語（CUDA）で書き換える

GPU計算の概要、NVIDIA GPU への移植方法についての比較は、下記の動画を参照ください。

- [GPUコンピューティングは難しいのか？](https://www.youtube.com/watch?v=pK-gllheNXE&t=22s)
- [An Introduction To GPU Programming Models](https://youtu.be/GKXG7OFTLzc?t=461)
- [A Deep Dive into the Latest HPC Software](https://www.nvidia.com/en-us/on-demand/session/gtcfall22-a41133/)
- [NVIDIA Deep Learning Institute: An Even Easier Introduction to CUDA](https://learn.nvidia.com/courses/course-detail?course_id=course-v1:DLI+T-AC-01+V1)

## GPUへの移植方法の特徴

### GPU対応ライブラリを用いる

特定のアルゴリズムについては、GPU向け（CUDA）ライブラリが存在しています。既存のCPU向けライブラリをGPU向けライブラリに入れ替えると、その部分のみですが、GPUで高速に計算されます。GPU向けライブラリとしては、下記のようなものがあります。OpenACCやCUDAとの併用は可能で、ライブラリで対応できない部分は、OpenACC などでGPU化する必要があります。

- [cuFFT](https://docs.nvidia.com/cuda/cufft/index.html)：フーリエ変換
- [cuBLAS](https://docs.nvidia.com/cuda/cublas/index.html)：行列演算（密行列）
- [cuSPARSE](https://docs.nvidia.com/cuda/cusparse/index.html)：行列演算（疎行列）
- [cuRAND](https://docs.nvidia.com/cuda/curand/index.html)：乱数生成
- [Thrust](https://docs.nvidia.com/cuda/thrust/index.html)：C++テンプレートライブラリ（STLベース）

### 既存のCPUコードに指示文（ディレクティブ）を挿入する

#### OpenACC

OpenACC指示文（ディレクティブ）を既存のCPUコードに挿入することで、GPU向けコードをコンパイラが作成します。OpenACCは並列計算を容易に実現するもので、GPUを主な対象とし、C言語とFortranに対応しています。指示文で、ループ部分のGPU化、CPUとGPU間のデータ通信の挿入などを行うことができます。また、CUDAでは記述が複雑なリダクション計算もOpenACCでは簡単に記述できます。既存CPUコードに指示文を挿入するため、CPU向けのコードと共通化することが可能です。

詳細は下記のセンター開催の講習会資料と録画を参照ください。

- [GPUプログラミング入門（過去開催時の資料・録画映像）](https://www.cc.u-tokyo.ac.jp/events/lectures/226/#lecture-doc)

#### OpenMPのtarget指示文

OpenMP指示文（ディレクティブ）は、主に共有メモリ型マルチプロセッサの並列プログラミングに利用されます。OpenMP 4.0からtarget指示文が導入され、並列処理をGPUなどアクセラレータへオフロードする機能が提供されています（GPU Offloading）。OpenACCと同様なことがOpenMPでも実現できるようになりました。しかし、現時点では、NVIDIA GPUでは、OpenMP GPU Offloading よりもOpenACCの方が高い性能がでる傾向にあります。

#### GPU向け指示文統合マクロSolomon

OpenACCとOpenMPのtarget指示文には機能・ドキュメントの豊富さおよびサポートされているGPUベンダーという観点でトレードオフがあります。
Solomon（Simple Off-LOading Macros Orchestrating multiple Notations）は、両指示文のインターフェースを統合したマクロ群であり、OpenACCとOpenMPのtarget指示文両方に対応したコードを簡単に実装できるようになります。

詳細は下記の参考資料を参照ください。

- [Miki & Hanawa (2024), IEEE Access, 12, 181644](https://doi.org/10.1109/ACCESS.2024.3509380)
- [Solomon: Simple Off-LOading Macros Orchestrating multiple Notations (GitHub)](https://github.com/ymiki-repo/solomon)

### 標準言語の並列処理でGPUを利用する

最近のC++やFortranを利用することで、標準言語自体が提供する機能によって並列処理することができます。C++ではC++17以降に提供される並列プログラミングの機能でコードを記述することで、それをGPUで実行することができます。また、Fortran では、Fortran 2008 で並列プログラミングをサポートするための機能が追加され、Fortran 2018 ではその機能を強化されています。CPUとGPUのどちらからもアクセス可能なメモリ（Unified memory）と併用するため、明示的なメモリ転送は不要です。このため、ソースコードは完全に標準言語のみで記述可能で、高い移植性を実現できます。性能としては、明示的なメモリ転送ができないこと、細かいGPU実行の制御ができないこと、コンパイラが開発途上であることから、OpenACCよりも若干性能が劣る傾向にあります。

詳細は下記の参考資料を参照ください。

- [Developing HPC Applications with Standard C++, Fortran, and Python (NVIDIA On-Demand)](https://www.nvidia.com/en-us/on-demand/session/gtcfall22-a41087/)
- [Developing Accelerated Code with Standard Language Parallelism (NVIDIA Technical Blog)](https://developer.nvidia.com/blog/developing-accelerated-code-with-standard-language-parallelism/)
- [Multi-GPU Programming with Standard Parallel C++, Part 1 (NVIDIA Technical Blog)](https://developer.nvidia.com/blog/multi-gpu-programming-with-standard-parallel-c-part-1/)
- [Multi-GPU Programming with Standard Parallel C++, Part 2 (NVIDIA Technical Blog)](https://developer.nvidia.com/blog/multi-gpu-programming-with-standard-parallel-c-part-2/)
- [Using Fortran Standard Parallel Programming for GPU Acceleration (NVIDIA Technical Blog)](https://developer.nvidia.com/blog/using-fortran-standard-parallel-programming-for-gpu-acceleration/)
- [Why Standards-Based Parallel Programming Should be in Your HPC Toolbox (HPCWire)](https://www.hpcwire.com/2022/09/05/why-standards-based-parallel-programming-should-be-in-your-hpc-toolbox/)
- [Leveraging Standards-Based Parallel Programming in HPC Applications (HPCWire)](https://www.hpcwire.com/2022/10/03/leveraging-standards-based-parallel-programming-in-hpc-applications/)
- [New C++ Sender Library Enables Portable Asynchrony (HPCWire)](https://www.hpcwire.com/2022/12/05/new-c-sender-library-enables-portable-asynchrony/)

### CUDAで書き換える

CUDAはNVIDIA GPU向けの開発環境および言語であり、C++言語を拡張した CUDA C++ と Fortran を拡張した CUDA Fortran があります。ユーザがゼロからGPU向けコードを記述する必要があります。柔軟性が高く、高性能なコードを作成可能ですが、既存のCPUコードからの書き換え量は多く、生産性は高くありません。CUDAコードはCPU向けのコードとは別にメンテナンスする必要があり、コードの保守の観点でもコストが高いです。

## GPU移行例の紹介

各種GPU化手法での実装例を追加していきます。

[N体計算](https://github.com/ymiki-repo/nbody)に関しては、コードに加えて性能の情報もまとまっており、どのGPU化手法を用いるかの比較検討にお使いいただけます。

[共役勾配法](https://github.com/kazuya-yamazaki/CG_on_GPU)に関しては、現時点では専らOpenACCでのコード例を掲載しています。

| 実装例 | 元の言語 | OpenACC | OpenMP | 標準言語の並列処理 | CUDA | Solomon |
| :----: | :----: | :----: | :----: | :----: | :----: | :----: |
| [N体計算](https://github.com/ymiki-repo/nbody) | C++ | OK | OK | OK | OK | OK |
| [共役勾配法](https://github.com/kazuya-yamazaki/CG_on_GPU) | C・FORTRAN | OK | - | - | - | - |

# GPU移行実践

## 講習会

東京大学情報基盤センターでは、GPUプログラミングに関する講習会を定期的に開催しております。受講料無料のオンライン講習会であり、過去開催時の資料や録画映像もご参照いただけます。

詳細は下記をご参照ください。

- [GPUプログラミング入門（過去開催時の資料・録画映像）](https://www.cc.u-tokyo.ac.jp/events/lectures/226/#lecture-doc)
- [OpenACCとMPIによるマルチGPUプログラミング入門（過去開催時の資料・録画映像）](https://www.cc.u-tokyo.ac.jp/events/lectures/228/#lecture-doc)
- [お試しアカウント付き並列プログラミング講習会](https://www.cc.u-tokyo.ac.jp/events/lectures/)

## ミニキャンプ

東京大学情報基盤センターでは、既存のCPUシミュレーションコードをCUDA、OpenACC、ライブラリでGPU化したり、既存の単体GPUコードを複数GPUコードにすることなどに取り組む「GPUミニキャンプ」を定期的に開催しております。皆様のCPUコードのGPU移植をサポートするため、2022年12月以降は頻度を増やして開催いたします。本ミニキャンプはZoomによるオンライン開催またはハイブリッド開催を予定しており、受講料は無料です。

これまでに参加チームによって取り組まれた課題の例です。
- 密度汎関数理論に基づく第一原理電子状態計算ソフトウェアOpenMXのGPU化とベンチマーク
- ⾮静⼒学数値海洋モデルkinacoのGPU化
- FMOプログラムABINIT-MPのGPU対応
- ⼤規模並列有限要素法構造解析ソフトウェアFrontISTRのGPU化

詳細は下記をご参照ください。

- [JCAHPC Open Hackathon（2025年1月31日、2月3日、2月10日、2月17日）](https://www.cc.u-tokyo.ac.jp/events/lectures/239/)
- [お試しアカウント付き並列プログラミング講習会](https://www.cc.u-tokyo.ac.jp/events/lectures/)

## GPU移行相談会

東京大学情報基盤センターでは、Miyabiの利用開始に向け、GPU移行に関する様々な疑問をGPU計算に実際に取り組んでいる研究者や技術者（チューター）と直接相談できる相談会を定期的に開催します。

- 既存のCPUコードをGPU化する適切な方法がわからない。
- GPU向けライブラリを紹介してほしい。
- 性能を上げるためのプロファイラの利用方法を聞きたい。
- 複数のGPUを利用した計算方法について相談したい。
- そもそもGPU計算がよくわからない。

など、どんなことでもお気軽にご相談ください。具体的な相談内容がなくても、相談会の様子を知るための参加も歓迎です。
本相談会はZoomによるオンライン開催を予定しており、事前申込制で、参加料は無料です。

GPU移行相談会の開催日と相談できるチューターの参加予定は下記サイトをご覧ください。チューターの専門分野によって、相談可能な（得意としている）言語やツールが異なります。参加される回を決める際にご利用ください。なお、チューターの参加予定は随時更新されています。

- [GPU移行相談会開催日と参加予定チューター](https://docs.google.com/spreadsheets/u/1/d/e/2PACX-1vR7-akq3aRkdSmhiq7L-Rk34BnqXFU3CRfUDkCu10lEWvwyWkkVz_ob1Q77zMX7184OrvxkFT921Ks5/pubhtml)

相談会に参加を希望される方は、下記の参加申込フォームへ必要項目を入力してお申し込みください。

- [GPU移行相談会 参加申込フォーム](https://regist.cc.u-tokyo.ac.jp/entry10/form.html)
