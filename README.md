# MIMIC-IV 真实 ICU 医疗数据 ETL 清洗项目

基于 **MIMIC-IV（重症监护医学信息数据库）** 真实脱敏 ICU 数据，使用 **Kettle（PDI）9.4 + MySQL** 完成完整的 ETL 清洗流程：数据抽取 → 清洗转换 → 装载入仓 → 质量评估。

## 项目背景

MIMIC-IV 是麻省理工与贝斯以色列女执事医疗中心公开的真实重症监护数据库（已脱敏）。本项目从 hosp 模块抽取 **275 条就诊记录的就诊 / 诊断 / 检验 / 用药** 4 张业务表（共约 12 万行）数据，进行清洗后入仓，输出 "患者就诊宽表" 和 "数据质量报告"，还原真实企业 ETL 开发流程。

## 技术栈



* **ETL 工具**：Kettle 9.4（PDI，图形化转换开发）

* **数据库**：MySQL 9.7（mimic\_dw 库）

* **数据规模**：原始数据约 12 万行，清洗后脏数据率从 **11% 降至 0.8%**

## ETL 任务流（6 个转换，全部在 Kettle GUI 亲手搭建）



| 任务 | 源表                         | 目标表                  | 清洗规则                                              | 结果                |
| -- | -------------------------- | -------------------- | ------------------------------------------------- | ----------------- |
| A  | admissions（275 行）          | admissions\_clean    | 语言字段 `?` → `UNKNOWN` 值映射                          | 275 行             |
| B  | diagnoses\_icd             | diagnosis\_clean     | **ICD 编码补小数点**（如 `4019`→`401.9`，第 4 位前加点）         | 4506 行            |
| C  | labevents + d\_labitems    | lab\_clean           | 过滤非数值 / 负值 / 缺 hadm\_id；**LEFT JOIN 字典表补检验项目名**   | 69548 行           |
| D  | prescriptions + admissions | medication\_clean    | 过滤 `stoptime<starttime` 时间逻辑错误（753 条）、给药途径缺失；外键校验 | 17328 行           |
| E  | 上述 4 张清洗表                  | patient\_visit\_wide | 多表聚合构建**患者就诊宽表**（16 字段，含住院时长、诊断 / 检验 / 用药计数）      | 275 行             |
| F  | 全部清洗表                      | quality\_report      | UNION ALL 统计各表质量分                                 | 5 行 |

## 核心成果指标



* **admissions\_clean**：275 条就诊记录，语言值映射清洗 0 遗漏

* **diagnosis\_clean**：4506 条诊断，ICD 编码补点 100% 正确（样本：401.9/E78.5/I10.）

* **lab\_clean**：69548 条检验记录，非数值 / 负值 / 缺关联全部过滤，检验名关联率 100%

* **medication\_clean**：17328 条用药记录，0 条时间逻辑错误、0 条途径缺失

* **patient\_visit\_wide**：275 行宽表，平均住院时长 164.5 小时，死亡率 5.5%

* **quality_report**：输出 5 张表数据质量报告

## 关键技术难点与解决



1. **笛卡尔积爆炸**：宽表首版 "三表直接 JOIN+GROUP BY" 产生千万级中间结果卡死。

   → **优化方案**：改为三个子查询**各自先按 hadm\_id GROUP BY 聚合，再 LEFT JOIN**，275 行秒出。

2. **Kettle 表输出 "未接收到任何字段"**：Kettle GUI 建表按钮失灵（预览正常、重连无效）。

   → **绕过方案**：直接在 MySQL 建好目标表，Kettle 表输出组件直接写入。


4. **业务数据校验**：用药记录中 753 条 `stop_time < start_time` 逻辑错误被过滤，。

## 文件结构



```
mimic_etl_project/
├── README.md                # 项目说明（本文件）
└── kettle/                  # Kettle 转换文件（.ktr，Kettle 9.4 可直接打开运行）
    ├── task1_patient.ktr       # 患者主表抽取
    ├── task2_visit.ktr         # 就诊记录抽取
    ├── task3_diagnosis.ktr     # 诊断数据抽取
    ├── task4_labtest.ktr       # 检验数据抽取
    ├── task5_medication.ktr    # 用药数据抽取
    ├── task6_wide.ktr          # 患者就诊宽表构建
    ├── task7_quality.ktr       # 数据质量评估
    ├── taskA_admissions.ktr    # admissions 清洗转换（语言值映射）
    ├── taskB_diagnosis.ktr     # 诊断清洗（ICD 编码补小数点）
    ├── taskC_labevent.ktr      # 检验清洗（过滤+字典表关联）
    └── taskF_quality.ktr       # 质量分统计
```

## 数据来源

MIMIC-IV v2.x（PhysioNet 公开数据集，需完成 CITI 培训认证后申请下载）：[https://physionet.org/content/mimiciv/](https://physionet.org/content/mimiciv/)
