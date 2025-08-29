## [[Intel]] [[Core Ultra]] 7 265Kによる[[Fedora]] Desktop 42上での[[AI環境]]構築

### 序論：[[Arrow Lake]]アーキテクチャと[[AI PC]]パラダイム

Intel Core Ultra 7 265Kプロセッサの登場は、デスクトップコンピューティングにおける重要な転換点を示します。このプロセッサは、単なる中央処理装置（CPU）ではなく、[[ヘテロジニアス]]（異種混合）コンピューティングのために設計されたSystem-on-a-Chip（[[SoC]]）です 1。[[Intel 4]]および[[TSMC N3B]]プロセス技術を基盤とする「[[Arrow Lake]]」アーキテクチャを採用し、「[[Lion Cove]]」[[P-core]]（高性能コア）と「[[Skymont]]」[[E-core]]（高効率コア）を組み合わせたハイブリッド設計を特徴としています 1。特筆すべきアーキテクチャ上の変更点として、P-coreからハイパースレッディングが廃止されたことが挙げられます。これにより、タスクスケジューリングの効率化は、オペレーティングシステムと密に連携する[[Intel Thread Director]]に、より大きく依存することになります 4。

この新しい設計思想の中核をなすのが、[[AIワークロード]]を処理するために最適化された3つの異なるエンジンです。これまでの[[CPU]]中心のモデルから、複数の専門化された計算ユニットを協調させる[[マルチエンジンSoCモデル]]への移行は、[[AIアプリケーション]]の性能を最大化するための新しいアプローチを要求します。開発者やパワーユーザーは、各エンジンの特性を理解し、ワークロードを戦略的に割り当てるという、システムアーキテクトとしての役割を担う必要があります。

### 3つのAIエンジンの定義

Core Ultra 7 265Kは、それぞれが異なる特性を持つ3つのAIエンジンを搭載しており、これらを統合的に活用することで、これまでにないレベルの性能と電力効率を実現します。

1. [[CPU]]（P-core/E-core）: Intel Deep Learning Boost（[[DL Boost]]）や[[AVX2命令セット]]をサポートするP-coreとE-coreは、汎用的かつ低遅延が求められるAIタスクに適しています。対話型のアプリケーションや、単一の推論を可能な限り高速に実行する必要がある場合にその能力を発揮します 5。
2. [[ iGPU]]（Intel® [[Arc]] Graphics）: 統合されたArc [[Xe-LPG]] Graphics 64EUは、単なるディスプレイ出力装置ではありません。[[Xe-core]]を搭載した強力な並列処理エンジンであり、バッチ推論やモデルのトレーニングにおいて重要な役割を果たす[[行列演算]]を高速に実行できます。これは、AIワークロードにおいて相当な[[TOPS]]（Tera Operations Per Second）性能を提供します 2。
3. [[NPU]]（[[Intel AI Boost]]）: ニューラル・プロセッシング・ユニット（NPU）は、持続的かつ電力効率に優れた推論処理に特化した低消費電力アクセラレータです。ビデオ会議中の背景ぼかしやノイズ抑制（[[GNA 3.5]]）といった常時稼働するAI機能や、バックグラウンドで継続的に実行されるAIタスクを担当し、CPUとGPUを他の高負荷な処理から解放します 5。
### 本レポートの目的

本ガイドは、この先進的なハードウェアのポテンシャルを、最新の[[カーネル]]とソフトウェアパッケージを備え、新しいハードウェアへの対応力に優れたLinuxディストリビューションである[[Fedora]] Workstation 42上で完全に引き出すための、決定的な手順を提供することを目的とします。BIOS設定の確認から始まり、OSのインストール、ドライバー、計算ランタイム、[[oneAPI]]ツールキット、そして最終的には[[PyTorch]]や[[Ollama]]といったアプリケーションレベルの[[AIフレームワーク]]に至るまで、ソフトウェアスタック全体を網羅的に解説します。

---

**表1: Core Ultra 7 265K AIエンジンの仕様**

| エンジン                  | 主要技術                          | コア/ユニット数               | 最大周波数   | ピークINT8 TOPS | 主なユースケース             |
| --------------------- | ----------------------------- | ---------------------- | ------- | ------------ | -------------------- |
| CPU (P-Core + E-Core) | Intel DL Boost, AVX2          | 8 P-Cores + 12 E-Cores | 5.5 GHz | ~12 TOPS     | 低遅延推論、汎用AIタスク        |
| iGPU                  | Intel® Arc™ Graphics (Xe-LPG) | 64 EUs (4 Xe-cores)    | 2.0 GHz | ~8 TOPS      | バッチ推論、モデルトレーニング、画像生成 |
| NPU                   | Intel® AI Boost               | NPU 3 (4-NPU cores)    | N/A     | ~13 TOPS     | 持続的な低電力推論、AI支援機能     |

出典: 2

注: TOPS値は構成やワークロードによって変動する可能性があり、上記は公表されている仕様に基づく参考値です。プロセッサ全体の合計ピークTOPSは約33です 6。

---

## セクション1：システム基盤と前提条件の設定

AIワークロードに最適化された環境を構築するための第一歩は、OSがハードウェアの能力を最大限に引き出せるよう、堅牢なシステム基盤を確立することです。これには、OSインストール前のBIOS/UEFI設定の確認、最新のOSとカーネルの導入、そしてコンパイルに必要な開発ツールの準備が含まれます。特にArrow Lakeのような最先端のハードウェアでは、Fedoraのような先進的なLinuxディストリビューションの選択が、単なる好みではなく機能的な要件となります。古いカーネルを持つディストリビューションでは、NPUや最新のiGPU機能をサポートするドライバーモジュールが欠けている可能性が高く、ハードウェアのポテンシャルを十分に活用できません。

### BIOS/UEFIの検証

OSをインストールする前に、システムのBIOS/UEFI設定を確認し、AIおよびグラフィックス性能に不可欠な機能を有効化することが極めて重要です。

- **Resizable BAR (ReBAR)の有効化**: Intel Arc GPUの性能を最大限に引き出すためには、[[Resizable BAR]]（[[ReBAR]]）の有効化が必須条件です。この機能により、CPUがGPUのビデオメモリ全体に一度にアクセスできるようになり、データ転送のボトルネックが解消されます。BIOS/UEFI設定メニュー（通常は「Advanced」または「PCI Subsystem Settings」内）で、このオプションが「Enabled」または「Auto」に設定されていることを確認してください 8。
    

### Fedora 42 Workstationのインストール

1. **インストール**: Fedora Workstation 42の公式イメージを使用して、標準的なインストールを実行します。
    
2. **システムアップデート**: インストール完了後、最初に実行すべき最も重要な作業は、システム全体のアップデートです。これにより、最新のLinuxカーネル、システムライブラリ、セキュリティパッチが適用され、新しいハードウェアへの対応が確実になります。ターミナルを開き、以下のコマンドを実行します。
    
    Bash
    
    ```
    sudo dnf upgrade --refresh
    ```
    

### カーネルの検証

システムのアップデート後、Arrow Lakeプロセッサの各コンポーネントがカーネルによって正しく認識されているかを確認します。これは、後のドライバーインストールの問題を未然に防ぐための重要な診断ステップです。

1. **カーネルバージョンの確認**: Fedora 42は、Arrow Lakeをサポートするのに十分な新しいカーネルを搭載しているはずです。現在のバージョンを確認します。
    
    Bash
    
    ```
    uname -r
    ```
    
2. **カーネルモジュールの初期化確認**: カーネルの起動ログ（`dmesg`）を調べて、iGPU（`i915`）とNPU（`ivpu`）のドライバーモジュールがエラーなくロードされているかを確認します。
    
    Bash
    
    ```
    # iGPUの初期化ログを確認
    dmesg | grep i915
    
    # NPUの初期化ログを確認
    dmesg | grep ivpu
    ```
    
    これらのコマンドでエラーメッセージが表示されず、デバイスが正常に初期化された旨のログが出力されれば、OSの基盤は正しくハードウェアを認識しています 10。
    

### 必須開発ツールのインストール

後のセクションでドライバーやライブラリをソースからビルドしたり、AIアプリケーションをコンパイルしたりするために、基本的な開発ツールをインストールしておく必要があります。

Bash

```
sudo dnf groupinstall "Development Tools"
sudo dnf install cmake kernel-devel
```

このコマンドにより、`gcc`、`g++`、`make`、`git`といった基本的なツールと、カーネルモジュールのビルドに必要な`kernel-devel`、多くのプロジェクトでビルドシステムとして利用される`cmake`がインストールされます 13。

---

**表2: 主要ソフトウェアコンポーネントとインストールソース**

| コンポーネント                | 目的                       | 主要なインストール元            | 主要なパッケージ/方法                                        |
| ---------------------- | ------------------------ | --------------------- | -------------------------------------------------- |
| カーネルドライバー (i915, ivpu) | ハードウェアの基本制御              | Fedoraカーネル            | `kernel` (システムアップデートで提供)                           |
| GPU計算ランタイム             | GPUでのOpenCL/Level Zero実行 | Fedora公式リポジトリ         | `intel-compute-runtime`, `intel-level-zero`        |
| NPUユーザー空間ドライバー         | NPUでのLevel Zero実行        | COPRリポジトリ             | `intel-npu-level-zero`                             |
| oneAPIツールキット           | 高性能計算ライブラリとコンパイラ         | Intel公式DNFリポジトリ       | `intel-oneapi-dpcpp-cpp`, `intel-oneapi-mkl-devel` |
| PyTorch + XPU          | AIフレームワーク (GPU対応)        | 専用Pipインデックス           | `pip install torch --index-url...`                 |
| Ollama (IPEX-LLM版)     | ローカルLLM実行環境 (GPU最適化)     | GitHubリリースからの直接ダウンロード | `Ollama Portable Zip`                              |

出典: 15

---

## セクション2：Arc iGPUの高性能計算有効化

Core Ultra 7 265Kに統合されたArcグラフィックスは、AIワークロードにおける主要な並列処理エンジンです。このiGPUを計算タスクに利用するためには、グラフィックス表示用のドライバーだけでなく、[[OpenCL]]や[[oneAPI Level Zero]]といった計算APIをサポートする専用のランタイムをインストールする必要があります。幸いなことに、Intel GPUの計算サポートはLinuxエコシステムで成熟しており、必要なコンポーネントはFedoraの公式リポジトリから直接入手できます。これは、iGPUがプラットフォームのコア機能として認識されていることを示しており、セットアッププロセスを大幅に簡素化します。

### LinuxにおけるIntel GPU計算スタック

Intel GPUで計算処理を行うためのソフトウェアスタックは、複数のレイヤーで構成されています。

1. **カーネルドライバー (`i915`)**: ハードウェアを直接制御する最も低レベルのコンポーネント。Fedoraカーネルに標準で含まれています 10。
    
2. [[Mesa]]: 主に[[OpenGL]]や[[Vulkan]]といった[[グラフィックスAPI]]の実装を提供します。
    
3. [[Intel Compute Runtime]]: OpenCLやoneAPI Level Zeroといった、グラフィックスではなく汎用計算（[[GPGPU]]）のためのAPIを提供するユーザー空間のドライバーです。AIフレームワークがGPUの計算能力を利用するために不可欠なレイヤーです 15。
    

### 計算ランタイムのインストール

AIアプリケーションがiGPUの計算リソースにアクセスできるようにするため、Fedoraの公式リポジトリからIntel Compute Runtimeとその関連パッケージをインストールします。これには、oneAPIの低レベルインターフェースであるLevel Zeroと、より広く使われているOpenCLの両方の実装が含まれます。

Bash

```
sudo dnf install intel-compute-runtime intel-level-zero intel-opencl intel-level-zero-devel
```

このコマンドでインストールされる`intel-level-zero`パッケージは、後述する[[PyTorch XPU]]バックエンドや[[OpenVINO]]といった高レベルフレームワークがGPUと通信するための基盤となります 15。

### oneAPI Level Zeroの概要

oneAPI Level Zeroは、Intelの計算ハードウェア（CPU, GPU, アクセラレータ）をプログラミングするための、現代的で低レベルなAPIです。ハードウェアへの直接的なアクセス（[[direct-to-metal]]）を提供し、高レベルのランタイムやライブラリが必要とする細やかな制御を可能にします。このAPIは、[[SYCL]]のような抽象化されたプログラミングモデルのバックエンドとして機能し、ハードウェアに依存しないコードの記述をサポートします。この環境において、Level ZeroはiGPUとNPUの両方に対する統一されたインターフェースの基礎を形成します 24。

### `xpu-smi`によるモニタリング

[[xpu-smi]]（Intel XPU System Management Interface）は、NVIDIAの[[nvidia-smi]]に相当するコマンドラインツールで、IntelのGPUの状態を監視・管理するために不可欠です。これを使用することで、GPUの利用率、消費電力、温度、メモリ使用量などをリアルタイムで確認でき、AIワークロードが意図通りにGPUで実行されているかを検証できます。

#### インストール

`xpu-smi`は、Intelの公式GPUドライバーリポジトリに含まれており、RHELなどのディストリビューションではサポートされていますが、Fedoraの公式リポジトリには含まれていません 29。このため、ソースコードから手動でビルドする必要があります。これは、ユーザーが直面する可能性のある課題ですが、以下の手順で解決できます。

1. **ソースコードのクローン**:
    
    Bash
    
    ```
    git clone https://github.com/intel/xpumanager.git
    cd xpumanager
    ```
    
2. **依存関係のインストール**: ビルドに必要なライブラリをインストールします。
    
    Bash
    
    ```
    sudo dnf install cmake make gcc-c++ libuuid-devel ncurses-devel json-c-devel libcurl-devel openssl-devel
    ```
    
3. **ビルドとインストール**:
    
    Bash
    
    ```
    mkdir build
    cd build
    cmake..
    make -j $(nproc)
    sudo make install
    ```
    

この手順により、`xpu-smi`がシステムにインストールされます 29。

#### 使用方法

インストール後、以下のコマンドでGPUの状態を確認できます。

- **デバイス一覧の表示**:
    
    Bash
    
    ```
    xpu-smi discovery
    ```
    
- **GPU統計情報のリアルタイム監視**:
    
    Bash
    
    ```
    # 1秒ごとに情報を更新
    xpu-smi stats -d 0 -w 1
    ```
    

このツールは、後のセクションでPyTorchやOllamaのワークロードを実行する際に、GPUが実際に活用されていることを確認するための重要な手段となります。

## セクション3：ニューラル・プロセッシング・ユニット（NPU）の有効化

Core Ultra 7 265Kが搭載するNPU（Intel AI Boost）は、持続的なAI推論タスクを低消費電力で実行するために設計された専用アクセラレータです。このNPUをLinux上で活用するには、iGPUとは異なる独自のソフトウェアスタックが必要です。このスタックは、カーネルレベルのドライバー、ハードウェアを初期化するためのファームウェア、そしてアプリケーションがNPUと通信するためのユーザー空間ドライバーから構成されます。

Intelの公式サポートは主にUbuntuを対象としていますが、活発なオープンソースコミュニティの活動により、FedoraユーザーもCOPR（Cool Other Package Repo）を通じてこれらの機能を利用できます。ただし、この方法はコミュニティ主導であるため、iGPUのセットアップに比べて実験的な側面が強く、エコシステムの成熟度の違いを理解した上で進める必要があります。

### LinuxにおけるNPUソフトウェアスタック

NPUを有効化するためのソフトウェアコンポーネントは以下の通りです。

1. [[ivpu]]カーネルドライバー: NPUハードウェアを制御するカーネルモジュール。Linuxカーネルのバージョン6.6以降で基本的なサポートが開始され、新しいバージョンで機能が拡充されています。Fedora 42の最新カーネルには、このドライバーが含まれているはずです 7。
    
2. **デバイスファームウェア**: NPUを動作させるためのマイクロコード。通常は`linux-firmware`パッケージに含まれていますが、最新の機能を利用するには、より新しいバージョンのファームウェアが必要になる場合があります。
    
3. **ユーザー空間ドライバー**: oneAPI Level Zero APIをNPU向けに実装したライブラリ。アプリケーション（例: OpenVINO）がNPUの計算リソースを利用するためのインターフェースを提供します 34。
    

### COPRリポジトリを利用したインストール

FedoraでNPUを有効化するための最も現実的で推奨される方法は、コミュニティによって維持されているCOPRリポジトリを利用することです。特に、Alexander Lent氏が管理する`xanderlent/intel-npu-driver`リポジトリは、Fedoraユーザーにとって重要なリソースとなっています 16。

1. **COPRリポジトリの有効化**: 以下のコマンドを実行して、DNFがこのリポジトリからパッケージを取得できるようにします。
    
    Bash
    
    ```
    sudo dnf copr enable xanderlent/intel-npu-driver -y
    ```
    
2. **NPUドライバーとファームウェアのインストール**: リポジトリを有効化した後、NPUの動作に不可欠なパッケージをインストールします。`intel-npu-firmware-upstream`は最新のファームウェアを提供し、`intel-npu-level-zero`はユーザー空間のLevel Zeroドライバーを提供します。
    
    Bash
    
    ```
    sudo dnf install intel-npu-firmware-upstream intel-npu-level-zero
    ```
    
    検証用に、テストアプリケーションを含む`intel-npu-level-zero-validation`パッケージをインストールすることも推奨されます 16。
    

### 代替案：Snapパッケージ

Canonicalが維持管理するSnapパッケージ（`intel-npu-driver`）も利用可能な選択肢の一つです。Snapは依存関係をすべて内包しているため、ディストリビューション間の互換性が高いという利点があります。しかし、Fedora環境ではCOPRを利用する方が、よりネイティブなパッケージ管理との親和性が高く、一般的に推奨されます 36。

### 検証と権限設定

ドライバーのインストール後、NPUがシステムによって正しく認識され、一般ユーザーがアクセスできる状態になっているかを確認する必要があります。

1. **カーネルログの確認**: `dmesg`コマンドを使用して、`ivpu`ドライバーがファームウェアを正常にロードし、デバイスを初期化したことを確認します。
    
    Bash
    
    ```
    dmesg | grep -iE 'ivpu|intel_vpu'
    ```
    
    成功を示すメッセージが表示されるはずです 12。
    
2. **デバイスノードの存在確認**: NPUはアクセラレータデバイスとしてシステムに認識され、`/dev/accel/`以下にデバイスファイルが作成されます。
    
    Bash
    
    ```
    ls /dev/accel/
    ```
    
    `accel0`のようなデバイスが表示されることを確認します 33。
    
3. **ユーザー権限の設定（重要）**: デフォルトでは、NPUデバイスへのアクセスはrootユーザーに限定されています。一般ユーザーがAIアプリケーションからNPUを利用できるようにするには、そのユーザーを`render`グループに追加する必要があります。これは見落とされがちな重要なステップです。
    
    Bash
    
    ```
    sudo usermod -aG render $USER
    ```
    
    このコマンドを実行した後、変更を有効にするために一度ログアウトし、再度ログインする必要があります 7。これにより、アプリケーションが
    
    `sudo`なしでNPUデバイスにアクセスできるようになります。
    

## セクション4：oneAPIエコシステムの導入

oneAPIは、Intelの多様なコンピューティングアーキテクチャ（CPU、GPU、FPGA、その他のアクセラレータ）にわたる統一されたプログラミングモデルを提供する、オープンな業界標準イニシアチブです。その中核をなすoneAPIツールキットには、高性能なコンパイラ、ライブラリ、解析ツールが含まれており、ヘテロジニアスなハードウェアの能力を最大限に引き出すために不可欠です。

IntelはDNF/YUMリポジトリを通じてoneAPIパッケージを提供していますが、Fedoraは公式にはサポート対象外です。そのため、最新のFedoraディストリビューションにインストールする際には、リポジトリのGPGキーやパッケージの署名に関する問題が発生する可能性があります。これらの技術的な問題を回避するには、単に公式ドキュメントに従うだけでは不十分で、専門的な知識に基づいた設定変更が必要となります。

### Intel DNFリポジトリの設定

まず、Intelの公式oneAPIリポジトリをシステムのDNF設定に追加します。

1. **GPGキーのインポート**:
    
    Bash
    
    ```
    sudo rpm --import https://yum.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
    ```
    
2. **リポジトリファイルの作成**: `/etc/yum.repos.d/oneAPI.repo`という名前で新しいファイルを作成し、以下の内容を記述します。
    
    Ini, TOML
    
    ```
    [oneAPI]
    name=Intel® oneAPI repository
    baseurl=https://yum.repos.intel.com/oneapi
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://yum.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
    ```
    

### Fedora固有の問題への対処

Fedora 38以降、RPMパッケージ管理システムの暗号化ライブラリが変更されたことに伴い、IntelのoneAPIリポジトリとの間に互換性の問題が生じることが報告されています 17。これらの問題を解決するため、以下の専門的な対処法を適用します。

1. **リポジトリ優先度の設定**: Fedoraの公式リポジトリには、oneAPIの一部のライブラリ（例：`oneTBB`）がすでにパッケージとして含まれています。Intelのリポジトリと競合した場合に、システムとの整合性が高いFedoraのパッケージが優先されるように、`oneAPI.repo`ファイルに優先度設定を追加します。DNFのデフォルト優先度は99であり、数値が小さいほど優先度が高いため、100以上の値を設定します。
    
    `/etc/yum.repos.d/oneAPI.repo`ファイルに以下の行を追加します。
    
    Ini, TOML
    
    ```
    priority=150
    ```
    
    これにより、ライブラリのパスが予期せぬ場所（`/opt/intel/oneapi`配下）にインストールされ、アプリケーションが共有ライブラリを見つけられなくなる問題を回避できます 17。
    
2. **GPGチェックの回避策（必要な場合のみ）**: パッケージのインストール中にGPGキーや署名に関するエラー（例: `invalid OpenPGP signature`）が発生した場合、一時的な回避策としてGPGチェックを無効化する必要があります。
    
    `/etc/yum.repos.d/oneAPI.repo`ファイルの`gpgcheck`と`repo_gpgcheck`を`0`に変更します。
    
    Ini, TOML
    
    ```
    gpgcheck=0
    repo_gpgcheck=0
    ```
    
    **警告**: この設定はリポジトリのパッケージ署名検証を無効にするため、セキュリティリスクを伴います。問題が解決された新しいバージョンのツールキットがリリースされた際には、この設定を元に戻すことが推奨されます 17。
    

### oneAPI Base Toolkitのインストール

oneAPIツールキットは非常に巨大なため、すべてをインストールするのではなく、本ガイドの目的に必要なコンポーネントのみを選択してインストールします。PyTorchのXPUバックエンドやOpenVINOの動作には、主にDPC++/C++コンパイラ、oneMKL（Math Kernel Library）、oneCCL（Collective Communications Library）が必要です。

Bash

```
sudo dnf install intel-oneapi-dpcpp-cpp intel-oneapi-mkl-devel intel-oneapi-ccl-devel
```

このコマンドにより、AIワークロードのコンパイルと実行に必要な中核的なライブラリがインストールされます 41。

### 環境変数の設定

oneAPIツールキットのコンパイラやライブラリを使用するには、それらのパスを環境変数に設定する必要があります。Intelは、このプロセスを簡素化するためのスクリプトを提供しています。

ターミナルでoneAPI関連の作業を行う前に、必ず以下のコマンドを実行して環境を初期化してください。

Bash

```
source /opt/intel/oneapi/setvars.sh
```

このコマンドをシェルの設定ファイル（例: `~/.bashrc`）に追加しておくと、新しいターミナルを開くたびに自動的に環境が設定されるため便利です 42。

## セクション5：PyTorch環境におけるTransformerワークロードの高速化

PyTorchは、今日のAI開発において最も広く利用されているフレームワークの一つです。Intelは、自社のGPU（XPU）上でPyTorchのパフォーマンスを最大化するため、専用のバックエンドと拡張ライブラリを提供しています。しかし、これらのコンポーネントは標準的なPyPI（Python Package Index）には含まれておらず、特別なURLを指定してインストールする必要があります。この依存関係の断片化は、セットアップを複雑にする要因であり、正確なインストールコマンドの実行が成功の鍵となります。

### Python環境のセットアップ

システム全体を汚染せず、プロジェクトごとに依存関係を管理するために、専用のPython仮想環境を作成することを強く推奨します。

Bash

```
# 仮想環境を作成
python3 -m venv intel_ai_env

# 仮想環境をアクティベート
source intel_ai_env/bin/activate
```

以降の`pip`コマンドは、すべてこのアクティブ化された環境内で実行します。

### Intel GPU（XPU）対応PyTorchのインストール

標準の`pip install torch`ではCPU版がインストールされてしまいます。Intel GPUサポート（XPU）を含むPyTorchをインストールするには、Intelが提供する専用のインデックスURLを指定する必要があります。

Bash

```
pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/xpu
```

このコマンドにより、`to('xpu')`デバイス指定子を認識し、計算グラフをIntel GPUにオフロードできるPyTorchの特別ビルドがインストールされます 9。

### IPEXとOptimum Intelのインストール

PyTorchの性能をさらに引き出すために、Intel Extension for PyTorch（IPEX）と、Hugging Face Transformersライブラリとの連携を簡素化するOptimum Intelをインストールします。

1. Intel Extension for PyTorch (IPEX)のインストール:
    
    IPEXは、自動混合精度（AMP）やグラフ最適化など、Intelハードウェアに特化した最適化を提供します。これもまた、専用のインデックスURLからインストールする必要があります。
    
    Bash
    
    ```
    pip3 install intel-extension-for-pytorch --extra-index-url https://pytorch-extension.intel.com/release-whl/stable/xpu/us/
    ```
    
    9
    
2. Optimum Intelのインストール:
    
    Optimum Intelは、Hugging FaceのモデルをIPEXで最適化し、XPUデバイスに配置するプロセスを数行のコードで実現する高レベルなライブラリです。
    
    Bash
    
    ```
    pip3 install optimum[intel]
    ```
    
    43
    

### 実践的なコード例

以下のPythonスクリプトは、Hugging Faceの`pipeline`を使用して感情分析タスクを実行し、CPUでのベースライン性能と、Optimum Intelを使ってiGPUで高速化した性能を比較する方法を示しています。

Python

```
import torch
import time
from transformers import pipeline
from optimum.intel import inference_mode

# 0. デバイスの確認
if torch.xpu.is_available():
    print("Intel XPU (GPU) is available.")
    device = "xpu:0"
else:
    print("XPU not found. Running on CPU.")
    device = "cpu"

# 1. パイプラインの準備
model_name = "distilbert-base-uncased-finetuned-sst-2-english"
text = "Fedora 42 with the Core Ultra 7 265K is an amazing platform for AI development."

# 2. ベースライン：CPUでの推論
print("\n--- Running on CPU ---")
cpu_pipe = pipeline("sentiment-analysis", model=model_name, device=-1) # device=-1 for CPU
start_time = time.time()
result_cpu = cpu_pipe(text)
end_time = time.time()
print(f"Result: {result_cpu}")
print(f"CPU execution time: {end_time - start_time:.4f} seconds")

# 3. 高速化：iGPU (XPU)での推論
if device.startswith("xpu"):
    print("\n--- Running on iGPU (XPU) with Optimum Intel ---")
    gpu_pipe = pipeline("sentiment-analysis", model=model_name) # No device specified yet

    # inference_modeがモデルをXPUに配置し、BF16などの最適化を自動適用
    with inference_mode(gpu_pipe, dtype=torch.bfloat16, jit=True) as opt_pipe:
        print(f"Model is on device: {opt_pipe.model.device}")
        start_time = time.time()
        result_gpu = opt_pipe(text)
        end_time = time.time()
        print(f"Result: {result_gpu}")
        print(f"GPU execution time: {end_time - start_time:.4f} seconds")
```

#### 実行と検証

このスクリプトを実行する際、別のターミナルで`xpu-smi stats -w 1`を実行しておきます。スクリプトの「Running on iGPU (XPU)」セクションが実行されると、`xpu-smi`の出力でGPU利用率（`GPU Util`）が上昇するのが確認できるはずです。これにより、計算が正しくiGPUにオフロードされ、高速化が実現していることを視覚的に検証できます 50。

## セクション6：最適化されたOllamaによる高スループットなローカルLLM

Ollamaは、大規模言語モデル（LLM）をローカル環境で手軽に実行するための強力なツールです。標準のOllamaでもGPUサポートは存在しますが、Intelのハードウェア（特にiGPU）の性能を最大限に引き出すには、Intelの最適化ライブラリと深く統合された特別なビルドを使用することが最も効果的です。この複雑なソフトウェアスタックをユーザーが手動で構築する手間を省くため、IntelのIPEX-LLMプロジェクトは「Ollama Portable Zip」を提供しています。これは、必要なライブラリや依存関係をすべて内包した自己完結型のパッケージであり、OllamaをIntel GPU上で実行するための、最も簡単で推奨される方法です。

このアプローチは、これまでのセクションで解説したドライバー、ランタイム、oneAPIツールキットの手動インストールプロセスを完全に迂回します。これは、プラットフォームの複雑さを抽象化し、エンドユーザーにシンプルな体験を提供するための戦略的なソリューションです。特定の目的（Ollamaの実行）のためには、より直接的で効率的なこの方法を選択することが賢明です。

### IPEX-LLM "Ollama Portable Zip" の導入

IPEX-LLMチームが提供するこのポータブル版Ollamaは、Intel GPU上での実行に必要なoneAPIランタイムとIPEX-LLMバックエンドをすべて同梱しています。ユーザーは数ステップで、最適化されたLLM推論環境を構築できます 20。

1. ダウンロード:
    
    IPEX-LLMのGitHubリリースページから、最新のLinux用Ollamaポータブルパッケージ（ollama-ipex-llm-....-ubuntu.tgzのような名前のファイル）をダウンロードします。
    
    - ([https://github.com/intel/ipex-llm/releases](https://github.com/intel/ipex-llm/releases))
        
2. 展開:
    
    ダウンロードしたファイルを適切なディレクトリに展開します。
    
    Bash
    
    ```
    tar -xvf ollama-ipex-llm-*.tgz
    cd ollama-ipex-llm-*
    ```
    
3. Ollamaサーバーの起動:
    
    展開されたディレクトリ内にある起動スクリプトを実行して、Ollamaサーバーをバックグラウンドで起動します。
    

./start-ollama.sh &

```

```
このスクリプトが、必要な環境変数を設定し、Intel GPUバックエンドを使用するOllamaサーバーを起動します。
```

4. モデルのダウンロードと実行:
    
    サーバーが起動したら、別のターミナルからollamaコマンドラインツールを使用してモデルをダウンロードし、対話を開始できます。
    
    Bash
    
    ```
    # Llama 3.2 3Bモデルをダウンロードして実行
    ```
    

./ollama run llama3.2

```

```
初回実行時にはモデルのダウンロードが開始され、完了するとプロンプトが表示されます。これで、ローカルLLMとの対話を開始できます [54]。
```

### 実行の検証

Ollamaが実際にiGPUを使用して推論を実行していることを確認するのは非常に重要です。`xpu-smi`ツールを使用して、リアルタイムでGPUの活動を監視します。

1. Ollamaでモデルとの対話を開始する前に、別のターミナルを開き、以下のコマンドを実行します。
    
    Bash
    
    ```
    xpu-smi stats -w 1
    ```
    
    これにより、1秒ごとにGPUの統計情報（利用率、電力、温度など）が表示されます。
    
2. Ollamaのプロンプトに質問を入力し、Enterキーを押して推論を開始します。
    
3. `xpu-smi`を実行しているターミナルに注目してください。Ollamaが応答を生成している間、`GPU Util`のパーセンテージが大幅に上昇するはずです。応答の生成が完了すると、利用率は再び低い値に戻ります。
    

このGPU利用率の変動は、LLMの計算負荷がCPUではなくiGPUに正しくオフロードされ、ハードウェアアクセラレーションが有効に機能していることの明確な証拠となります。

## 結論：統一された高性能AIプラットフォーム

本ガイドで詳述した手順を経て、Intel Core Ultra 7 265KプロセッサとFedora 42 Workstationを基盤とする、強力で柔軟なAI開発・実行環境が構築されました。このシステムは、単一のハードウェア上でCPU、iGPU、NPUという3つの異なるAIエンジンがすべて有効化され、それぞれがAIフレームワークからアクセス可能な状態にあります。これは、現代のAI PCが目指すヘテロジニアスコンピューティングの理念を具現化したものです。

### 戦略的なワークロードの割り当て

構築された環境の真価は、タスクの特性に応じて最適なエンジンを選択し、システム全体のスループットと効率を最大化する能力にあります。以下に、各エンジンの戦略的な活用指針を示します。

- NPU (Intel AI Boost):
    
    低消費電力で持続的な推論処理に最適です。リアルタイムの音声認識やノイズ抑制、あるいはビデオ会議での背景効果など、常にバックグラウンドで動作し続けるようなタスクに適しています。これらのワークロードをNPUにオフロードすることで、CPUとGPUの貴重なリソースをより要求の厳しいタスクのために温存できます 6。
    
- iGPU (Intel Arc Graphics):
    
    このプラットフォームにおけるAIワークロードの主力です。高い並列処理能力を活かし、バッチ推論、Ollamaを用いたローカルLLMの実行、Stable Diffusionのような画像生成モデル、さらには小規模なモデルのファインチューニングやトレーニングといった高スループットが求められるタスクに最も適しています。本ガイドで構築したOllama環境は、このiGPUの能力を直接活用する好例です 51。
    
- CPU (P-core & E-core):
    
    並列化が困難なタスク、単一の推論における最小の遅延が求められる場合、またはアプリケーションの他の部分とのデータ局所性が重要な場合に依然として最適な選択肢です。特に、ユーザーとの対話に即座に応答する必要があるアプリケーションのロジック部分でAI推論を行う際にその価値を発揮します 43。
    

### OpenVINOの統一的な役割と将来展望

本ガイドでは、PyTorchにはIPEX、OllamaにはIPEX-LLMといった特定のバックエンドを利用しましたが、Intelのソフトウェアエコシステム全体を見渡すと、[[OpenVINO]]ツールキット**がこれらの異種混合エンジンを統一的に扱うための究極的な抽象化レイヤーとして位置づけられています。

OpenVINOは、開発者が一度モデルを最適化すれば、実行時に利用可能なハードウェア（CPU、GPU、NPU）を自動的に検出し、最も適切なデバイスにワークロードをインテリジェントにディスパッチする能力を持っています。これにより、開発者はハードウェアごとの複雑な実装を意識することなく、アプリケーションのロジックに集中できます 55。

現在、OllamaのようなフレームワークにおけるOpenVINOの統合は進行中ですが、将来的には`ollama run my-model --device openvino`のようなシンプルなコマンドで、モデルが自動的に最適なエンジン（例えば、最初のプロンプト処理はGPUで、トークン生成はNPUで、といった高度な連携も）で実行される未来が期待されます。今回構築した環境は、そのような次世代のAIアプリケーションが登場した際に、その能力を即座に活用できる先進的なプラットフォームと言えるでしょう。