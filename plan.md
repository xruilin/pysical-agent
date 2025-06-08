# 解决物理问题的智能体构建计划 (v2)

## 1. 核心理念与目标

本计划旨在构建一个能够理解和解决复杂物理问题的智能体。核心思路是结合多模态大语言模型（如 `qvq-plus` 用于视觉理解，`qwen-plus` 用于核心推理）与外部计算工具（Python, Sympy, Wolfram Mathematica）和一个持续迭代的知识库，以实现高精度、可解释的物理问题求解。

**目标用户**: 学生、教师、物理爱好者。
**核心能力**:
*   准确理解包含文本、公式和图像的物理问题。
*   生成多种求解思路并进行评估。
*   执行必要的数值和符号计算。
*   提供清晰的解题步骤和可视化结果。
*   通过与知识库的交互和用户反馈持续学习和改进。

## 2. 系统架构与模块

智能体将主要由以下几个模块构成：

### 2.1. 问题理解与表征模块
*   **输入**: 物理题目（文本描述 + 图像/示意图）。
*   **模型**: `qvq-plus` (或类似的多模态模型)。
*   **功能**:
    *   **文本提取**: 准确提取题目中的所有文字信息。
    *   **公式识别**: 识别并抽取出 LaTeX 或其他标准格式的数学公式。
    *   **图像/示意图分析**:
        *   识别图像中的关键物体、部件及其标签。
        *   理解物体间的空间关系、连接方式、接触状态。
        *   识别物理状态的表征，如速度矢量、力矢量、运动轨迹示意等。
        *   提取图中标注的已知量和待求量。
*   **输出**: 结构化的JSON对象，包含：
    *   `problem_text`: 原始或清洗后的题目文本。
    *   `extracted_formulas`: 公式列表 (e.g., `["E=mc^2", "F=ma"]`)。
    *   `image_elements`: 图像中识别出的物体及其属性 (e.g., `[{"label": "block_A", "mass": "m1", "on_surface": "inclined_plane"}, ...]`)。
    *   `image_relationships`: 物体间的关系 (e.g., `["block_A contacts block_B", "spring_1 connects to block_A"]`)。
    *   `diagram_annotations`: 图中标注的符号及其含义。
*   **优化**: 针对物理场景优化提示工程，确保 `qvq-plus` 输出的准确性和全面性。考虑对输出进行校验和可能的澄清环节。

### 2.2. 核心推理与求解模块
*   **输入**: 结构化的问题表征。
*   **模型**: `qwen-plus` (或类似的强文本推理模型)。
*   **功能**:
    1.  **知识库检索 (RAG)**:
        *   根据问题表征，从知识库中检索相关的物理概念、核心公式、基本定理、解题模板和相似已解题目。
    2.  **求解策略生成**:
        *   基于检索到的知识和问题本身，生成多种可能的求解路径/策略（例如，牛顿定律、能量守恒、动量守恒、运动学分析等）。
        *   允许模型自主探索或通过预设的元提示（meta-prompts）引导生成不同类型的解法。
    3.  **步骤分解与执行**:
        *   将每个求解策略分解为详细的逻辑步骤和计算任务。
        *   **调用计算工具**:
            *   **Python (Sympy, Numpy, Scipy)**: 对于需要精确数值计算或复杂符号运算（如解方程组、求导、积分）的步骤，LLM生成相应的Python代码，并在沙箱环境中执行，获取结果。
            *   **Wolfram Mathematica (可选)**: 对于更高级的符号计算或特定函数库需求。
        *   LLM负责解释计算工具的输出，并将其整合回推理链条。
*   **输出**: 每种求解策略的完整推理过程、中间步骤、调用的代码、计算结果和最终答案。

### 2.3. 结果评估与整合模块
*   **输入**: 来自核心推理模块的多种求解方案。
*   **模型**: 可由 `qwen-plus` 承担，或设计专门的评估模型/规则集。
*   **功能**:
    *   **多维度评估**:
        *   **正确性**:
            *   **物理一致性**: 单位检查、量纲分析。
            *   **逻辑自洽性**: 步骤间推导是否正确，有无矛盾。
            *   **边界/特例检验**: 在极端或简化条件下，结果是否符合物理直觉。
            *   **与已知条件吻合度**: 结果是否与题目所有给定条件兼容。
        *   **效率与简洁性**: 解法步骤是否最少，有无更直接的路径。
        *   **可解释性**: 推理过程是否清晰、易于理解。
        *   **知识库对齐**: 与知识库中高质量解法的相似度和一致性。
    *   **方案选择/融合**:
        *   选出最优解法，或对多个有效解法进行综合展示。
        *   如果所有解法均存在问题，系统应能分析可能的原因，并触发重试（调整策略）或向用户请求澄清。
*   **输出**: 最终推荐的解答（含详细步骤）、对不同解法的评估摘要、置信度评分。

### 2.4. 知识库模块
*   **目标**: 存储和管理物理知识、解题经验，支持智能体的学习和迭代。
*   **内容**:
    *   **物理概念库**: 定义、公式、适用条件、相关定理。
    *   **问题库**: 结构化存储的物理题目及其标准表征。
    *   **解法库**: 经过验证的高质量解题方案，包括详细步骤、涉及的公式、使用的计算代码、解题技巧和常见错误。
    *   **评估反馈库**: 存储对不同解法的评估结果和用户反馈。
*   **技术选型**: 考虑使用矢量数据库（如FAISS, Milvus）进行相似题目/解法检索，图数据库（如Neo4j）管理概念间关系，或结合结构化文件存储。
*   **更新机制**:
    *   **自动入库**: 经评估模块验证为高质量的新解法自动加入。
    *   **失败案例记录**: 记录不成功的尝试及其原因，用于避免重复错误和优化策略。
    *   **用户反馈整合**: 将用户对解答的评价（正确性、清晰度等）纳入，用于调整评估标准和解法权重。

### 2.5. 输出与可视化模块
*   **输入**: 最终的解答和解题过程。
*   **功能**:
    *   **格式化输出**: 以Markdown、LaTeX等易读格式呈现解题步骤、公式、最终答案。
    *   **代码展示**: 清晰展示用于计算的Python或其他代码。
    *   **可视化生成**:
        *   调用Python库（Matplotlib, Seaborn）或Wolfram Mathematica生成图表（如受力图、运动轨迹图、数据图）。
        *   LLM可生成相应的可视化代码。
*   **输出**: 用户友好的、包含文本、公式、代码和可视化图表的完整解答。


## 3. 迭代开发与学习流程

### 3.1 整体架构概览

首先展示智能体的整体工作流程：

```mermaid
flowchart LR
    A["📥 输入处理"] --> B["🧠 推理求解"] --> C["⚖️ 评估整合"] --> D["📤 结果输出"]
    
    %% 核心支撑
    KB[("📚 知识库")]
    B <--> KB
    C -.->|"学习更新"| KB
    
    %% 反馈循环
    D --> F{"💬 反馈"}
    F -.->|"优化"| KB
    F -->|"完成"| END["✅"]
    
    %% 错误处理
    A -.->|"失败重试"| A
    B -.->|"策略调整"| B
    C -.->|"重新评估"| B
    
    classDef main fill:#4fc3f7,stroke:#0288d1,stroke-width:3px,color:#000
    classDef support fill:#81c784,stroke:#388e3c,stroke-width:2px,color:#000
    classDef feedback fill:#ffb74d,stroke:#f57c00,stroke-width:1px,stroke-dasharray:3 3,color:#000
    
    class A,B,C,D main
    class KB,END support
    class F feedback
```

### 3.2 输入处理模块详解

```mermaid
flowchart TD
    INPUT["📥 物理问题输入<br/>(文本 + 图像)"] --> PARSE["🔍 多模态解析<br/>(qvq-plus)"]
    
    subgraph "解析处理"
        PARSE --> T1["📝 文本提取"]
        PARSE --> T2["🧮 公式识别"] 
        PARSE --> T3["🖼️ 图像分析"]
        
        T1 --> R1["问题描述"]
        T2 --> R2["数学公式"] 
        T3 --> R3["物理要素<br/>• 物体关系<br/>• 标注信息"]
    end
    
    R1 --> JSON["📋 结构化表征<br/>(JSON)"]
    R2 --> JSON
    R3 --> JSON
    
    JSON --> VALIDATE{"✅ 校验完整性"}
    VALIDATE -->|"通过"| NEXT["➡️ 推理模块"]
    VALIDATE -.->|"失败"| INPUT
    
    classDef input fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef process fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000
    classDef output fill:#e8f5e8,stroke:#388e3c,stroke-width:2px,color:#000
    
    class INPUT,PARSE input
    class T1,T2,T3,VALIDATE process
    class R1,R2,R3,JSON,NEXT output
```

### 3.3 推理求解模块详解

```mermaid
flowchart TD
    START["📋 问题表征输入"] --> RETRIEVE["🔍 知识检索<br/>(RAG)"]
    
    subgraph KB_SEARCH ["知识库检索"]
        RETRIEVE --> K1["物理概念"]
        RETRIEVE --> K2["解题模板"]
        RETRIEVE --> K3["相似案例"]
    end
    
    K1 --> STRATEGY["💡 策略生成"]
    K2 --> STRATEGY
    K3 --> STRATEGY
    
    subgraph MULTI_STRATEGY ["多策略并行"]
        STRATEGY --> S1["🏃 运动分析<br/>• 牛顿定律<br/>• 运动学"]
        STRATEGY --> S2["⚡ 能量方法<br/>• 动能定理<br/>• 能量守恒"]
        STRATEGY --> S3["💥 动量分析<br/>• 动量守恒<br/>• 冲量定理"]
        
        S1 --> C1["🐍 Python计算"]
        S2 --> C2["🐍 Python计算"]
        S3 --> C3["🔬 Wolfram计算"]
        
        C1 --> R1["📊 方案1结果"]
        C2 --> R2["📊 方案2结果"]
        C3 --> R3["📊 方案3结果"]
    end
    
    R1 --> NEXT["➡️ 评估模块"]
    R2 --> NEXT
    R3 --> NEXT
    
    classDef input fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef knowledge fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    classDef strategy fill:#e8f5e8,stroke:#388e3c,stroke-width:2px,color:#000
    classDef compute fill:#fce4ec,stroke:#c2185b,stroke-width:2px,color:#000
    classDef output fill:#f1f8e9,stroke:#689f38,stroke-width:2px,color:#000
    
    class START,RETRIEVE input
    class K1,K2,K3 knowledge
    class STRATEGY,S1,S2,S3 strategy
    class C1,C2,C3 compute
    class R1,R2,R3,NEXT output
```

### 3.4 评估整合模块详解

```mermaid
flowchart TD
    RESULTS["📊 多方案结果输入"] --> EVALUATE["⚖️ 多维度评估"]
    
    subgraph EVAL_PROCESS ["评估维度"]
        EVALUATE --> E1["🔬 正确性检查<br/>• 单位量纲<br/>• 逻辑自洽<br/>• 边界条件"]
        EVALUATE --> E2["⚡ 效率评估<br/>• 步骤简洁性<br/>• 计算复杂度"]
        EVALUATE --> E3["📖 可解释性<br/>• 推理清晰度<br/>• 物理直觉"]
        
        E1 --> SCORE1["评分1"]
        E2 --> SCORE2["评分2"] 
        E3 --> SCORE3["评分3"]
    end
    
    SCORE1 --> INTEGRATE["🔄 方案整合"]
    SCORE2 --> INTEGRATE
    SCORE3 --> INTEGRATE
    
    INTEGRATE --> DECISION{"🎯 决策"}
    DECISION -->|"单一最优"| BEST["👑 最佳方案"]
    DECISION -->|"多方案展示"| MULTI["📚 综合展示"]
    DECISION -.->|"无满意解"| RETRY["🔄 重新求解"]
    
    BEST --> OUTPUT["➡️ 输出模块"]
    MULTI --> OUTPUT
    RETRY -.-> BACK["⬅️ 返回推理模块"]
    
    classDef input fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef eval fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    classDef integrate fill:#e8f5e8,stroke:#388e3c,stroke-width:3px,color:#000
    classDef output fill:#f1f8e9,stroke:#689f38,stroke-width:2px,color:#000
    classDef retry fill:#ffebee,stroke:#d32f2f,stroke-width:1px,stroke-dasharray:3 3,color:#000
    
    class RESULTS input
    class EVALUATE,E1,E2,E3,SCORE1,SCORE2,SCORE3 eval
    class INTEGRATE,DECISION integrate
    class BEST,MULTI,OUTPUT output
    class RETRY,BACK retry
```

### 3.5 输出与学习模块详解

```mermaid
flowchart TD
    INPUT_RESULT["📊 评估结果输入"] --> FORMAT["📝 格式化处理"]
    
    subgraph OUTPUT_GEN ["输出生成"]
        FORMAT --> F1["📄 步骤详解<br/>• Markdown格式<br/>• LaTeX公式"]
        FORMAT --> F2["💻 代码展示<br/>• Python代码<br/>• 计算过程"]
        FORMAT --> F3["📈 可视化<br/>• 图表生成<br/>• 物理示意图"]
        
        F1 --> O1["文字解答"]
        F2 --> O2["代码块"]
        F3 --> O3["图表文件"]
    end
    
    O1 --> COMBINE["🔗 内容整合"]
    O2 --> COMBINE
    O3 --> COMBINE
    
    COMBINE --> USER["👤 用户展示"]
    USER --> FEEDBACK{"💬 用户反馈"}
    
    subgraph LEARNING ["学习循环"]
        FEEDBACK -->|"正面反馈"| GOOD["✅ 优质案例"]
        FEEDBACK -->|"负面反馈"| BAD["❌ 问题记录"]
        FEEDBACK -->|"改进建议"| IMPROVE["🔧 优化建议"]
        
        GOOD --> KB_UPDATE["📚 知识库更新"]
        BAD --> KB_UPDATE
        IMPROVE --> KB_UPDATE
    end
    
    KB_UPDATE --> KB[("🧠 知识库")]
    FEEDBACK -->|"无反馈"| END["✅ 完成"]
    
    classDef input fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef process fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000
    classDef output fill:#e8f5e8,stroke:#388e3c,stroke-width:2px,color:#000
    classDef learning fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    classDef storage fill:#f1f8e9,stroke:#689f38,stroke-width:3px,color:#000
    
    class INPUT_RESULT input
    class FORMAT,F1,F2,F3,COMBINE process
    class O1,O2,O3,USER,END output
    class FEEDBACK,GOOD,BAD,IMPROVE,KB_UPDATE learning
    class KB storage
```

### 3.6 完整系统交互图

```mermaid
flowchart TD
    subgraph SYSTEM ["🏗️ 物理问题求解智能体"]
        M1["📥 输入处理"] --> M2["🧠 推理求解"]
        M2 --> M3["⚖️ 评估整合"] 
        M3 --> M4["📤 输出学习"]
        
        KB[("📚 知识库")]
        M2 <--> KB
        M3 -.-> KB
        M4 -.-> KB
    end
    
    USER["👤 用户"] --> M1
    M4 --> USER
    USER -.->|"反馈"| M4
    
    TOOLS["🔧 外部工具<br/>• Python/Sympy<br/>• Wolfram<br/>• 可视化库"] 
    M2 --> TOOLS
    TOOLS --> M3
    
    classDef main fill:#4fc3f7,stroke:#0288d1,stroke-width:3px,color:#000
    classDef support fill:#81c784,stroke:#388e3c,stroke-width:2px,color:#000
    classDef external fill:#ffb74d,stroke:#f57c00,stroke-width:2px,color:#000
    
    class M1,M2,M3,M4 main
    class KB,USER support
    class TOOLS external
```

这种分段细化的方法有以下优势：

1. **渐进式理解**：从整体概览到详细实现
2. **模块化设计**：每个组件都有清晰的职责边界
3. **易于开发**：可以按模块逐步实现和测试
4. **便于维护**：问题定位和优化更加精准
5. **灵活扩展**：可以独立升级某个模块而不影响整体

