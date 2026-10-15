# Rastreio e Estimativa de Profundidade em Vídeo

Sistema que, a partir de um vídeo (ou de imagens capturadas por webcam), **segmenta os objetos, estima a profundidade de cada um, identifica qual está mais próximo da câmera e o rastreia ao longo do tempo**, mantendo o mesmo ID quando o objeto sai e volta ao quadro.

Trabalho entregue em Google Colab: o notebook contém o código documentado e o relatório de evidências.

Integrantes:
| Nome | RM |
|------|----|
| Augusto Oliveira Codo de Sousa | RM562080 |
| Felipe de Oliveira Cabral | RM561720 |
| Gabriel Tonelli Avelino Dos Santos | RM564705 |
| Marcelo Roberto Maso Junior | RM562163 |
| Vinícius Adrian Siqueira de Oliveira | RM564962 |
| Sofia Bueris Netto de Souza | RM565818 |

## Abrir no Colab

1. Envie `rastreio_profundidade.ipynb` para o Google Drive (ou abra direto em [colab.research.google.com](https://colab.research.google.com) → Arquivo → Fazer upload de notebook).
2. Ative a GPU em **Ambiente de execução → Alterar tipo de ambiente de execução → GPU (T4)**.
3. Execute **Ambiente de execução → Executar tudo**.

> Dica: se o notebook estiver neste repositório do GitHub, o Colab também abre direto por `https://colab.research.google.com/github/<usuario>/<repositorio>/blob/main/rastreio_profundidade.ipynb`.

## Como funciona

```
 frame ──► Mask R-CNN ──► máscaras + caixas + classes
   │                              │
   └──► MiDaS small ──► mapa de profundidade relativa
                                  │  mediana do mapa dentro de cada máscara
                                  ▼
                proximidade de cada objeto (0 = longe, 1 = perto)
                                  │
                                  ▼
   Tracker: Kalman + Húngaro (IoU + cor HSV) + re-identificação
                                  │
                                  ▼
        Seletor de alvo: objeto mais próximo (histerese + paciência)
```

| Etapa | Técnica |
|---|---|
| Segmentação de instâncias | Mask R-CNN (ResNet-50 FPN v2, torchvision, pesos COCO) |
| Profundidade | MiDaS small (CNN), profundidade relativa (inversa) por frame |
| Associação | Algoritmo húngaro, custo = `0,6·(1−IoU) + 0,4·distância de cor` |
| Movimento | Filtro de Kalman de velocidade constante (centro, largura, altura) |
| Entrada e saída do quadro | Galeria de tracks perdidos + re-identificação por histograma HSV |
| Alvo | Objeto mais próximo, com histerese (`MARGEM_TROCA`) e paciência (`PACIENCIA`) |

A profundidade é **relativa**, não métrica: serve para ordenar quem está mais perto, não para medir distâncias em metros.

## Entrada de dados

Escolha em `FONTE` (seção 2 do notebook):

| Valor | O que faz |
|---|---|
| `exemplo` | Baixa um vídeo de exemplo (pessoa, bicicleta e carro) |
| `upload` | Pede para você enviar um vídeo seu |
| `pasta_de_imagens` | Lê frames `.jpg`/`.png` de uma pasta (ex.: capturas de webcam) |

O vídeo de exemplo tem poucos objetos simultâneos e nenhuma re-entrada. Para demonstrar bem o sistema, use um vídeo com **dois ou três objetos a distâncias diferentes** e pelo menos um **saindo e voltando ao quadro**.

## Parâmetros principais

Todos ficam em formulários do Colab na seção 2.

| Parâmetro | Função |
|---|---|
| `MAX_FRAMES`, `LARGURA` | Quantos frames processar e a largura de trabalho |
| `SCORE_MIN` | Confiança mínima da detecção |
| `CLASSES_INTERESSE` | Classes COCO aceitas, separadas por vírgula (vazio = todas) |
| `MAX_AGE`, `MIN_HITS` | Quantos frames um track sobrevive sem detecção e quantos acertos para confirmá-lo |
| `GATE` | Custo máximo aceito na associação |
| `MAX_LOST`, `REID_THR` | Tempo na galeria de perdidos e limiar de cor para reaproveitar um ID |
| `MARGEM_TROCA`, `PACIENCIA` | Histerese do alvo e frames de espera pelo alvo que saiu |

## Evidências geradas pelo notebook

- Vídeo anotado, com máscaras, IDs, trajetórias, alvo em vermelho e mini mapa de profundidade
- Quadros-chave lado a lado: original, profundidade e segmentação com tracking
- Proximidade de cada objeto ao longo do tempo e ID do alvo por frame
- Linha do tempo de presença de cada ID
- Relatório cronológico de entradas e saídas (`ENTROU`, `SAIU`, `VOLTOU`) e imagens antes/depois de cada re-entrada
- Ablação do tracker com e sem re-identificação (IDs únicos)
- Validação da profundidade: correlação de Spearman com pistas geométricas (posição vertical e área da caixa) e estabilidade temporal
- Tempo por etapa e FPS do sistema completo
- Resumo automático dos resultados e conclusões

## Resultados

_Preencher após executar com o seu vídeo: nº de objetos rastreados, re-identificações, correlação de profundidade e FPS (a célula "Resumo automático" do notebook imprime esses números)._

## Limitações

- Profundidade relativa e normalizada por frame, sem escala em metros.
- A re-identificação usa apenas cor (HSV): objetos da mesma classe com cores parecidas podem ser confundidos.
- Sem *ground truth* de profundidade, a validação é indireta.
- Os parâmetros dependem da cena e podem precisar de ajuste.

## Possíveis melhorias

Profundidade métrica (ZoeDepth, Depth Anything), embeddings de ReID aprendidos, associação em duas etapas (ByteTrack) e acompanhamento de múltiplos alvos.

## Referências

- He et al., *Mask R-CNN*, ICCV 2017.
- Ranftl et al., *Towards Robust Monocular Depth Estimation: Mixing Datasets for Zero-shot Cross-dataset Transfer* (MiDaS), TPAMI 2022.
- Bewley et al., *Simple Online and Realtime Tracking* (SORT), ICIP 2016.
- Kuhn, *The Hungarian Method for the Assignment Problem*, 1955.
