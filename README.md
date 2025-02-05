# deepseek-r1-openshift
deploying deepseek-r1 on openshift

DeepSeek Janus and DeepSeek-R1 are both large language models developed by DeepSeek, but they have different architectures and purposes:

## DeepSeek Janus
- Multi-modal model: Unlike DeepSeek-R1, which is purely text-based, Janus can process both text and images.
- Vision capabilities: It can understand and generate responses based on visual input, making it useful for applications like document understanding, image captioning, and multimodal reasoning.
- Performance: Since it integrates vision and language, it is expected to perform well in tasks requiring contextual understanding beyond just text.

## DeepSeek-R1
- Code-specialized model: DeepSeek-R1 is optimized for code generation and understanding, making it particularly useful for software development tasks.
- Architecture: Built as a dense transformer-based model, it focuses on handling programming-related queries, debugging, and code completion.
- Training focus: While it understands natural language, its primary strength is code comprehension across multiple programming languages.

|Feature|DeepSeek Janus|DeepSeek-R1|
|---|---|---|
|Modality|Text & Image|Text only|
|Specialization|General-purpose, multimodal reasoning|Code generation & understanding|
|Use Cases|Image analysis, document understanding, general AI tasks|Image analysis, document understanding, general AI tasks|

References:
[DeepSeek Janus Pro 7B On Prem deployment on Openshift via Validated Patterns](https://medium.com/@chris.xg.wang/deepseek-janus-pro-7b-on-prem-deployment-on-openshift-via-validated-patterns-14d6ff3eddf8)