# ncbi-genome-inventory（中文说明）

**一条命令整理某个类群在 NCBI 的全部全基因组：每个组装用了哪些原始数据、同一样品的线粒体、分类阶元，以及哪些物种正在测序。**

[English README](README.md)

以前要手工完成的流程：在 NCBI Datasets 网页勾选导出组装表，逐个 BioProject 去 SRA 查原始数据，判断哪些是 HiFi、哪些是 Hi-C，再找线粒体、到 WoRMS 匹配学名，最后在 Excel 里合并。本工具把这些步骤合成一条命令。下文所有示例都是 **环节动物门（Annelida）** 的真实运行结果（2026-10-28）。

![网页界面](docs/web_ui.png)

## 目录

- [功能一览](#功能一览)
- [安装](#安装)
- [快速上手](#快速上手)
- [使用方法](#使用方法)：命令行、网页版、更新
- [每个功能能得到什么（环节动物实例）](#每个功能能得到什么环节动物实例)
- [输出文件](#输出文件)
- [工作原理](#工作原理)
- [局限](#局限)
- [测试](#测试)

## 功能一览

| # | 功能 | 解决的问题 |
|---|---|---|
| 1 | **组装总表**：前 41 列与 NCBI Datasets 网页导出完全一致 | 这个类群有哪些基因组，质量如何？ |
| 2 | **分类阶元**：NCBI 门到种 + WoRMS 有效名 | 每个基因组属于哪个科，学名是否有效？ |
| 3 | **每个组装的原始数据**：HiFi、CLR、ONT、Hi-C、短读长 WGS、10x、RNA-seq、Iso-Seq，给出 run 号和数据量 | 这个基因组是用什么数据组装的？ |
| 4 | **是否同一个体**：根据 ENA 样本关系和 ToLID 判断 | Hi-C、RNA-seq 是否来自组装用的同一只动物？ |
| 5 | **每个组装一个线粒体，优先同一样品**；同名替代会醒目标注 | 这个基因组对应的线粒体是哪条？ |
| 6 | **正在测序的物种**（GoaT）：尚无 NCBI 组装的物种及其测序项目 | 这个类群接下来会出哪些基因组？ |
| 7 | **三个数据源交叉核对**（NCBI Datasets、Entrez、GoaT）+ QC 表 | 有没有遗漏或不一致？ |
| 8 | **增量更新** + 变更报告 | 和上次相比有什么新变化？ |
| 9 | **本地网页界面**：搜索、筛选、下载 | — |

只用 Python 标准库（Python ≥ 3.8）：不需要 pip 安装任何包，也不需要 `datasets`、`edirect`、`kingfisher`。

## 安装

```bash
git clone https://github.com/deyuanyang92-dev/ncbi-genome-inventory.git
cd ncbi-genome-inventory
python3 genome_inventory.py --help
```

可选：设置 NCBI API key，查询速度约快 3 倍。

```bash
export NCBI_API_KEY=你的key        # 在 https://www.ncbi.nlm.nih.gov/account/settings/ 申请
export NCBI_EMAIL=你的邮箱
```

## 快速上手

```bash
python3 genome_inventory.py Annelida -o Annelida_genome_inventory
```

运行时会逐步打印进度，下面是环节动物真实运行的结尾部分：

```text
[13:46:05] DONE
  assemblies (rows): 251
  species/taxa: 163
  SRA runs of these taxa: 7391
  runs linked to an assembly: 3148
  mitochondrial records: 240
  QC items: 145
  GoaT species in pipeline without NCBI assembly: 368
  assemblies with linked data type: HiFi 153, CLR 8, PacBio_unknown 1, ONT 19, Hi-C 169, short_read_WGS 92, 10x_linked 17, RNA-seq 120, Iso-Seq 3, Other 3
  mitogenome per assembly: same sample: in assembly 107; same sample: same BioSample 65; SUBSTITUTE ... 44; none 35
  linked run-assembly pairs by individual: same individual 3148; different individual 176; unknown 156
  -> Annelida_genome_inventory/Annelida_genome_inventory_20261028.xlsx
```

打开最后一行给出的 `.xlsx` 文件即可，各工作表的含义见[输出文件](#输出文件)。

用时（无 API key，实测）：一个科（如 Polynoidae，27 个组装）首次约 5 分钟；一个门（如环节动物）首次需要几十分钟。之后直接用缓存约 2 秒；`--update` 在 NCBI 没有新组装时约 15–30 秒。

## 使用方法

### 命令行

```bash
python3 genome_inventory.py 类群 [选项]
```

| 选项 | 含义 |
|---|---|
| `类群` | 类群名（`Annelida`、`Polynoidae`、`Platynereis`）或 NCBI taxid（`6340`） |
| `-o, --out 目录` | 输出目录（默认 `<类群>_genome_inventory`） |
| `--taxid ID` | 类群名有重名时用 taxid 指定（工具会列出候选） |
| `--update` | 重新检查 NCBI 和 GoaT 的组装名单，只为新物种、新样品补查原始数据、线粒体和分类阶元 |
| `--refresh` | 忽略缓存，全部重新抓取 |
| `--all-versions` | 同时列出被替代的旧版本组装（默认只要当前版本） |
| `--no-worms` | 不做 WoRMS 学名匹配 |
| `--no-goat` | 不查 GoaT（交叉核对和在测物种） |
| `--email`、`--api-key` | NCBI 身份（否则读环境变量 `$NCBI_EMAIL`、`$NCBI_API_KEY`） |

```bash
python3 genome_inventory.py Polynoidae                          # 一个科
python3 genome_inventory.py 6340 -o Annelida_out                # 用 taxid
python3 genome_inventory.py Annelida -o Annelida_out --update   # 以后更新
```

某一步因网络失败时，重新运行同一条命令即可，已完成的步骤会从 `<输出目录>/raw/` 缓存读取。

### 网页版

```bash
python3 webapp.py --data-dir genome_inventory_results       # 然后用浏览器打开 http://127.0.0.1:8765
```

1. 输入类群名；已有结果时勾选 **update**；点 **Run**，页面实时显示运行日志。
2. 运行完成后（或点击已有的结果文件夹），各表以标签页打开：assemblies、species、mitogenomes、qc、runs、changes、goat_pipeline。
3. 在筛选框输入科名、`SUBSTITUTE`、`different individual` 等即可过滤；点表头排序；取消 *key columns only* 可显示全部列。同名替代的线粒体用橙色标出。
4. 标签页上方是 Excel 和 TSV 文件的下载链接。

网页版调用的是同一个 `genome_inventory.py`，结果与命令行完全一致。服务只在本机（127.0.0.1）监听，并校验 Host、Origin 和每次启动生成的随机令牌。

### 更新

加 `--update` 重新运行（网页版勾选 update）。**Changes** 表会列出与上次结果相比的变化。运行前请先关闭 Excel，否则新结果会另存为 `..._HHMM.xlsx`。

## 每个功能能得到什么（环节动物实例）

### 1. 组装总表：与 NCBI 网页导出同列

**Assemblies** 表前 41 列的列名和顺序与 NCBI Datasets 网页导出完全一致（`Assembly Accession` … `CheckM contamination`），以前的表格和脚本可以直接沿用。所有组装都保留，包括同一物种的多套组装和备用单倍型组装。

| Assembly Accession | 物种 | 级别 | 发布日期 | 大小 (bp) | Contig N50 | Scaffold N50 | 测序技术 | 提交单位 |
|---|---|---|---|---|---|---|---|---|
| GCA_949317995.1 | Acholoe squamosa | Chromosome | 2023-04-04 | 1,481,470,071 | 5,523,906 | 84,155,211 | PacBio,Arima2 | Wellcome Sanger Institute |
| GCA_965276055.1 | Harmothoe areolata | Chromosome | 2025-05-02 | 1,663,694,339 | 20,708,742 | 88,728,759 | PacBio,Arima | Genoscope |
| GCA_026936325.1 | Platynereis dumerilii | Scaffold | 2022-12-16 | 1,469,601,079 | 4,288,053 | 54,798,709 | PacBio RSII; Illumina HiSeq | The Platynereis Community |

新增列包括：`representative_derived`（每个物种标一个代表性组装：核基因组 > 主组装 > RefSeq 参考 > 级别 > contig N50 > 最新）、`assembly_content_derived`（环节动物有 7 个"Complete Genome"组装其实只是一条约 16 kb 的线粒体）、`found_in_derived`（在哪些数据源中找到）。

### 2. 分类阶元：NCBI 与 WoRMS 并列

| 组装 | NCBI 科 | NCBI 种 | WoRMS 有效名 | WoRMS 状态 |
|---|---|---|---|---|
| GCA_949317995.1 | Polynoidae | Acholoe squamosa | Acholoe squamosa（AphiaID 146474） | accepted |
| GCA_963942565.1 | Magelonidae | Maea johnstoni | Magelona johnstoni（AphiaID 130269） | superseded combination |

第二行这类名称不一致会列入 QC 表；NCBI 原值不做修改。

### 3. 每个组装用了哪些原始数据

Assemblies 表对每种数据类型给出 `<类型>_runs`（run 数）、`<类型>_Gb`（数据量）、`<类型>_accessions`（run 号），并用 `data_types_derived` 汇总：

| 组装 | 物种 | data_types_derived | HiFi | Hi-C | RNA-seq | 关联依据 |
|---|---|---|---|---|---|---|
| GCA_949317995.1 | Acholoe squamosa | HiFi+Hi-C | 2 个 run，55.6 Gb（ERR10841326; ERR10841327） | 1 个 run，229.2 Gb（ERR10851523） | — | sibling_project |
| GCA_982089195.1 | Melinna palmata | HiFi+Hi-C+RNA-seq | 3 个 run，127.7 Gb | 1 个 run，146.5 Gb | 1 个 run，16.8 Gb | sibling_project |
| GCA_033239005.1 | Lamellibrachia satsuma | ONT+short_read_WGS+RNA-seq | — | — | 9 个 run，30.8 Gb | bioproject; biosample |

**Runs** 表中每个 run 都给出数据类型、判定规则和可信度：

| Run | 物种 | 测序仪 | 建库方案 | 类型 | 可信度 | 规则 |
|---|---|---|---|---|---|---|
| ERR10841326 | Acholoe squamosa | Sequel II | PacBio - HiFi | HiFi | high | PacBio + 关键词 HiFi |
| ERR10851523 | Acholoe squamosa | NovaSeq 6000 | Hi-C - Arima v2 | Hi-C | high | LibraryStrategy=Hi-C |
| ERR12876326 | Carpetania matritensis | PacBio RS II | — | CLR | medium | HiFi 出现之前的机型 |
| ERR13433269 | Alentia gelatinosa | Sequel IIe | — | HiFi | low | Sequel IIe、无关键词（推断） |

环节动物：Hi-C 关联到 169 个组装，HiFi 153 个，RNA-seq 120 个，ONT 19 个。

### 4. 是否同一个体

通过项目关联到的数据，不一定来自被组装的那只动物。`individual_derived` 列逐条判断：

| Run | 类型 | 物种 | individual_derived |
|---|---|---|---|
| ERR10841326 | HiFi | Acholoe squamosa | same individual: BioSample relation（样品衍生自组装样品） |
| ERR10851523 | Hi-C | Acholoe squamosa | same individual: BioSample relation |
| ERR16650963 | HiFi | Melinna palmata | different individual: ToLID wtMelPalm9 vs wtMelPalm2 |
| ERR16650975 | Hi-C | Melinna palmata | different individual: ToLID wtMelPalm1 vs wtMelPalm2 |

组装表给出 `same_individual_types_derived`（来自同一个体的数据类型）和 `other_individual_types_derived`（来自其他个体的数据类型）。*Acholoe* 的 HiFi 和 Hi-C 都来自参考个体；*Melinna* 的组装来自 wtMelPalm2，而 HiFi、Hi-C 分别来自 wtMelPalm9、wtMelPalm1。环节动物：同一个体 3,148 对，不同个体 176 对，无法判断 156 对。

### 5. 每个组装一个线粒体，优先同一样品

| 组装 | 物种 | mito_accession | 长度 (bp) | mito_source_derived | 备注 |
|---|---|---|---|---|---|
| GCA_949317995.1 | Acholoe squamosa | OX439053.1 | 15,641 | same sample: in assembly | |
| GCA_949316145.1 | Acholoe squamosa（备用单倍型） | OX439053.1 | 15,641 | same sample: same BioSample | |
| GCA_965119645.1 | Owenia fusiformis | OZ221967.1 | 18,801 | same sample: in assembly | 仅线粒体的组装，NCBI 误标为"Chromosome" |
| GCA_965276055.1 | Harmothoe areolata | CBDHZN010000007.1 | 77,074 | same sample: in assembly | 超过 30 kb，可能是多拷贝串联，需核实 |
| GCA_059996285.1 | Harmothoe imbricata | MK858187.1 | 15,694 | **SUBSTITUTE: same species name, different sample** | 2 条同名记录来自 2 个样品，身份未核实 |

只有找不到同一样品的记录时才用同名记录替代，并标注 SUBSTITUTE、列入 QC 表，因为 NCBI 上的鉴定无法保证正确。替代记录优先选原始 INSDC 记录，而不是 RefSeq 的 `NC_` 副本。环节动物：线粒体在组装内 107 个，同一 BioSample 65 个，同名替代 44 个，没有 35 个。

### 6. 正在测序的物种（GoaT）

**GoaT_pipeline** 表列出 [GoaT](https://goat.genomehubs.org) 报告为正在测序、但 NCBI 上还没有组装的物种。环节动物共 368 种，以 Nereididae 为例：

| 物种 | goat_sequencing_status | goat_projects |
|---|---|---|
| Hediste diversicolor | in_assembly | ATLASEA; DTOL |
| Nereis pelagica | in_assembly | ATLASEA; DTOL |
| Alitta succinea | data_generation | ATLASEA; CANBP; DTOL |
| Perinereis cultrifera | data_generation | ATLASEA; DTOL |
| Hediste sinesimplex | in_progress | ATLASEA |
| Neanthes rubicunda | sample_acquired | ATLASEA |

GoaT 测序进度从早到晚：`sample_collected`（已采样）→ `sample_acquired`（样品已送达）→ `data_generation`（正在产生数据）→ `in_assembly`（正在组装）→ `insdc_open`（组装已公开）→ `published`（已发表）；`in_progress` 表示项目只报了"进行中"、没说到哪一步。已有组装的物种，其 GoaT 状态写在 **Species** 表的 `goat_sequencing_status` 列。

也可以直接在浏览器里问 GoaT：

```text
https://goat.genomehubs.org/search?query=tax_tree(Annelida)%20AND%20tax_rank(species)%20AND%20sequencing_status&result=taxon&fields=sequencing_status,long_list
```

把 `Annelida` 换成任意类群；只看某一阶段时写成 `sequencing_status=in_assembly`。注意网页结果也包括已有组装的物种。

### 7. 交叉核对与 QC

NCBI Datasets、Entrez `assembly`、GoaT 三个来源的组装名单互相核对（环节动物：Datasets 252 个、Entrez 252 个完全一致，GoaT 的 250 个全部包含在内）。**QC** 表示例：

| 范围 | ID | 名称 | 说明 |
|---|---|---|---|
| mitogenome | GCA_059746215.1 | Tubifex tubifex | 没有组装同一样品的线粒体，用同名替代 MW690579.1 |
| assembly | GCA_043381215.1 | Platynereis dumerilii | 声明使用 PacBio，但找不到对应的 PacBio run |
| assembly | GCA_965119645.1 | Owenia fusiformis | 仅线粒体（< 1 Mb），不是核基因组 |
| worms | 1872991 | Maea johnstoni | WoRMS 状态为 superseded combination，有效名 Magelona johnstoni |

### 8. 变更报告

`--update` 之后，**Changes** 表会出现 `new assembly`（新增组装）、`no longer listed`（不再列出）、`field changed | Assembly Level | Scaffold → Chromosome`（字段变化）这类记录；没有变化时会注明。

## 输出文件

`<输出目录>/<类群>_genome_inventory_<日期>.xlsx`，同时输出同样内容的 `.tsv`：

| 工作表 | 每行是 | 主要内容 |
|---|---|---|
| Assemblies | 一个组装（GCA；对应的 GCF 在 `Assembly Paired Assembly Accession`） | NCBI 41 列、分类阶元、数据类型和 run 号、同一个体判定、线粒体 |
| Species | 一个物种 | 代表性组装、全部组装、数据类型、线粒体、GoaT 状态 |
| Runs | 一个 SRA run | RunInfo、建库方案、数据类型及规则和可信度、关联的组装、个体判定 |
| Mitogenomes | 一条线粒体记录 | 长度、拓扑、BioSample、与组装的关系 |
| QC | 一个问题 | 范围、ID、名称、说明 |
| Changes | 与上次结果的一处差异 | 变化类型、组装、字段、旧值、新值 |
| GoaT_pipeline | 一个正在测序、尚无 NCBI 组装的物种 | 纲、目、科、测序进度、测序项目 |
| Query | — | 参数、规则、统计 |

以 `_derived` 结尾的列由本工具计算，每列都有明确规则；其余列是 NCBI 原值，不做改动。`.xlsx` 中文本按纯文本保存；`.tsv` 中以 `= + - @` 开头的文本前面加了单引号，防止被表格软件当作公式执行。

## 工作原理

| 步骤 | 数据来源 |
|---|---|
| 组装名单和信息 | NCBI Datasets v2（NCBI 于 2024 年 6 月关闭 Assembly/Genome 网站，功能并入 Datasets）；与 Entrez `assembly` 库、GoaT 交叉核对 |
| 组装内的序列（找线粒体） | NCBI FTP 上的 `*_assembly_report.txt`（一次请求得到全部序列名）；失败时改用 Datasets `sequence_reports` |
| 原始数据 | SRA RunInfo + SRA XML（实验标题、设计、建库方案） |
| 是否同一个体 | ENA `related_sample_accession`（`derived_from`、`same_as`、`has_part`）和样品属性 `tolid` |
| 线粒体记录 | nuccore 按物种检索（10–100 kb）、按 BioSample 检索（不限长度） |
| 分类阶元 | NCBI Taxonomy；WoRMS `AphiaRecordsByMatchNames` |
| 正在测序的物种 | GoaT 接口（`sequencing_status`、`long_list`） |

**原始数据如何关联到组装**（只考虑同种或其亚种的 run）：`biosample`（同一 BioSample）> `bioproject`（同一项目或上下级项目）> `sibling_project`（两个项目有同一个直接上级）。最后一种是 Sanger/DToL/ERGA 的提交方式：*Acholoe squamosa* 的原始数据在 PRJEB59385，组装在 PRJEB60118，二者同属物种总项目 PRJEB59386。同种但没有这些关联的 run，只计入 Species 表。

**数据类型判定**（按顺序取第一个符合的；关键词取自实验标题、设计说明、文库名和建库方案）：

| 类型 | 规则 |
|---|---|
| Hi-C | LibraryStrategy = Hi-C，或含 Hi-C / Omni-C / Arima / Dovetail / proximity ligation 关键词（RNA 除外） |
| HiFi | PacBio + HiFi/CCS 关键词；Revio/Vega 机型；Sequel IIe 且无关键词（可信度低） |
| CLR | PacBio + CLR/subread 关键词；RS / RS II / Sequel 机型（可信度中） |
| PacBio_unknown | Sequel II 且无关键词（列入 QC） |
| ONT / ONT_RNA | Oxford Nanopore 基因组 / 转录组 |
| Iso-Seq / RNA-seq | PacBio 转录组 / 短读长转录组 |
| short_read_WGS | Illumina、BGI、DNBSEQ 等平台的 WGS |
| 10x_linked | 含 10x / Chromium / linked-read / stLFR / TELL-seq 关键词 |
| Other | 宏基因组、扩增子、RAD-seq 等 |

**线粒体优先级**：组装内（标为线粒体，或序列名含 mito/MT/chrM）> 与组装同一 BioSample 的记录 > 所关联原始数据 BioSample 的记录 > 同名替代（SUBSTITUTE）。同一级别内：原始 INSDC 记录 > 完整 > 环状 > ≥ 10 kb > 最新。

## 局限

- **未公开的原始数据无法关联。** 环节动物 195 个核基因组主组装中有 33 个没有公开原始数据（如 *Platynereis dumerilii*、早年 Sanger 测序的组装、尚未进入 SRA/ENA 的 2026 年新组装），QC 表逐个列出。
- **没有标注的线粒体找不到。** 组装里如果线粒体是一条普通命名的片段，需要序列比对（如 BLAST）才能识别，本工具不做。
- **同名替代的线粒体没有核实身份**，使用前请自行核对（如用 COI 与组装比对）。
- **GoaT 的测序进度由各项目自行上报**，可能滞后；"在测"不代表一定会发布。
- 数据类型判定依赖提交者填写的文字；不确定的判定标为低可信度或 `PacBio_unknown`。

## 测试

```bash
python3 -m pytest -q tests        # 离线运行，使用裁剪过的真实 NCBI 数据
```

## 致谢

数据来自 NCBI（Datasets、E-utilities、FTP）、ENA、GoaT / genomehubs 和 WoRMS。`ncbi.py`、`xlsx.py` 改编自 [yzz_taxonomy_workflow](https://github.com/deyuanyang92-dev/yzz_taxonomy_workflow) 的 `modules/sra`（该模块用于盘点某类群在 SRA 的全部 run）。本工具最初在 [Temp_scripts](https://github.com/deyuanyang92-dev/Temp_scripts) 中开发，现以本仓库为准。

## 许可证

尚未选定许可证。
