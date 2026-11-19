# AGENTS.md — Guia de Contexto e Regras do Projeto

Este arquivo define a arquitetura, os objetivos científicos, as restrições de hardware e as convenções operacionais deste repositório. Qualquer agente de CLI ou assistente de código operando neste diretório deve seguir estritamente estas diretrizes.

---

## 1. Objetivo do Projeto

Construir um repositório reprodutível para um **estudo comparativo de redes neurais** aplicadas à **detecção, rastreamento multi-objeto (MOT) e detecção de pontos-chave** em vídeos esportivos gravados em **4K (`3840x2160`)**.

O problema envolve dois desafios principais em cenas de ângulo aberto:
1. **Rastreamento de Jogadores (`bbox` + `MOT`):** Alvos que ocupam poucos pixels no quadro 4K, sujeitos a oclusões, cruzamentos e desfoque de movimento (*motion blur*).
2. **Detecção de Pontos-Chave do Campo (`keypoints`):** Localização geométrica precisa de interseções e marcações do campo.

---

## 2. Ambiente e Restrições de Hardware (Crítico)

* **Sistema Operacional:** Linux (local).
* **GPU:** NVIDIA **RTX 2060 (6 GB VRAM)**.
* **Regras de Execução para Scripts de Treino/Inferência:**
  * Nunca assumir mais de **6 GB de VRAM**.
  * Sempre habilitar precisão mista (`amp=True` / `FP16`) nos scripts de treino.
  * Para manter a visibilidade de jogadores pequenos em 4K, evitar redimensionar imagens para `640x640` quando possível; priorizar resoluções maiores (ex.: `imgsz=1280`) compensando com `batch_size` reduzido (`2` ou `4`) e *gradient accumulation*.
  * Para inferência em 4K sem estourar VRAM, suportar estratégias de fatiamento como **SAHI** (*Slicing Aided Hyper Inference*).

---

## 3. Arquitetura Híbrida de Armazenamento (GitHub + Hugging Face)

Este projeto separa **código/metadados** de **pesos binários**, mantendo exatamente a **mesma hierarquia e profundidade de pastas** em ambos os serviços:

* **GitHub (este repositório):** Armazena scripts (`.py`), arquivos de configuração de treino e rastreamento (`.yaml`, `.json`), citações bibliográficas (`cite.bib`, `referencias_gerais.bib`) e documentação (`README.md`, `AGENTS.md`, `catalogo_modelos.md`).
* **Hugging Face Hub (`<PENDENTE_CONFIGURAR_REPO_ID>`):** Armazena exclusivamente os arquivos pesados de pesos e modelos exportados (`.pt`, `.pth`, `.safetensors`, `.weights`, `.onnx`, `.h5`), já bloqueados no `.gitignore`.

### Estrutura de Diretórios Esperada

```text
.
├── AGENTS.md                       # Instruções para agentes de CLI (este arquivo)
├── README.md                       # Documentação geral e instruções de reprodução
├── .gitignore                      # Ignora pesos binários (*.pt, *.pth, *.weights, *.onnx, etc.)
├── requirements.txt                # Dependências e versões exatas do ambiente Python
├── catalogo_modelos.md             # Checklist de redes base, contextos treinados e métricas
├── referencias_gerais.bib          # Arquivo BibTeX consolidado de todas as arquiteturas
├── scripts/
│   ├── leitor_modelos.py           # Carrega arquiteturas e injeta pesos (local ou via HF)
│   └── sync_hf.py                  # Sincroniza pesos de 02_Finetuned/ espelhando a árvore no HF
├── 01_Base_Pretrained/             # 100% no GitHub (não sobe pesos para o HF)
│   └── <nome_arquitetura>/         # Ex: yolov11/, rtdetr_r50/, hrnet_w32/
│       ├── baixar_base.py          # Script para baixar arquitetura e pesos oficiais da fonte
│       └── cite.bib                # Referência BibTeX do artigo original
└── 02_Finetuned/                   # Configs no GitHub | Pesos retreinados espelhados no HF
    └── <contexto>/                 # Ex: rastreio_jogadores_4k/, keypoints_campo/
        └── <nome_arquitetura>/     # Ex: yolov11s/, rtdetr_r50/
            ├── config_treino.yaml  # [GitHub] Hiperparâmetros, resolução, época e seed
            ├── tracker.yaml        # [GitHub] Config do rastreador (ByteTrack/BoT-SORT), se aplicável
            └── <modelo_final>.pt   # [Hugging Face] Pesos retreinados (ignorado pelo Git)
