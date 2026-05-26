# Projeto: Gerador 3D a partir de Imagem (SaaS)

## Visão Geral

Construir uma aplicação web que recebe uma imagem 2D enviada pelo usuário e retorna um modelo 3D texturizado (formato `.glb`) renderizável no navegador. O sistema usa o modelo **Microsoft TRELLIS.2-4B** (image-to-3D, 4 bilhões de parâmetros) executado em GPU sob demanda, orquestrado por uma arquitetura serverless de baixo custo.

## Objetivo

Permitir que designers, desenvolvedores de jogos, artistas 3D e usuários casuais transformem fotos/ilustrações em assets 3D com materiais PBR (Base Color, Roughness, Metallic, Opacity) prontos para uso em engines como Unity, Unreal, Blender ou visualização web via three.js.

## Stack Técnica

### Frontend

- **Framework**: Next.js 14+ (App Router) hospedado em Cloudflare Pages
- **Renderização 3D**: three.js + React Three Fiber para preview do `.glb` no browser
- **Upload**: drag-and-drop com pré-visualização e remoção de fundo client-side (BRIA-RMBG-2.0 via Transformers.js, opcional)
- **UI**: Tailwind CSS + shadcn/ui

### Backend / Orquestração

- **Cloudflare Workers**: API REST para criar jobs, consultar status e receber webhooks do RunPod
- **Cloudflare D1** (SQLite): persistência de jobs, usuários, histórico de gerações
- **Cloudflare KV**: cache de status de jobs em tempo real
- **Cloudflare Queues**: fila de jobs com retry automático e rate limiting por usuário
- **Cloudflare R2**: armazenamento dos arquivos `.glb` e previews `.mp4` gerados (zero egress)

### Processamento GPU

- **RunPod Serverless** com GPU RTX 4090 (24GB VRAM) ou A100 80GB para resoluções maiores
- Container Docker customizado baseado em `nvidia/cuda:12.4-devel-ubuntu22.04`
- Modelo **TRELLIS.2-4B** baixado do Hugging Face e cacheado em Network Volume
- Handler Python que recebe imagem base64, executa o pipeline e devolve `.glb` + preview `.mp4`

### Autenticação e Pagamento

- **Clerk** ou **Auth.js** para login (Google, GitHub, email)
- **Stripe** para créditos pré-pagos ou assinatura mensal (ex: 50 gerações/mês)

## Fluxo de Funcionamento

1. Usuário faz upload de imagem no frontend
1. Frontend pré-processa (remove fundo, valida tamanho) e envia para Worker
1. Worker valida créditos do usuário, cria job no D1, enfileira no Queue, retorna `jobId`
1. Queue consumer dispara request assíncrono para RunPod Serverless
1. Frontend faz polling no endpoint `/status/:jobId` (ou usa SSE)
1. RunPod processa: imagem → O-Voxel → mesh → texturização PBR → `.glb`
1. Container faz upload do resultado direto pro R2 e chama webhook do Worker
1. Worker atualiza status do job para `completed` com URL do arquivo
1. Frontend detecta conclusão, renderiza preview 3D inline e oferece download

## Configurações de Resolução

|Tier  |Resolução|Tempo (4090)|Custo/geração|Plano         |
|------|---------|------------|-------------|--------------|
|Free  |512³     |~3-5s       |~$0.003      |3 gerações/dia|
|Pro   |1024³    |~17-25s     |~$0.015      |50/mês        |
|Studio|1536³    |~60-90s     |~$0.06       |200/mês + API |

## Funcionalidades Principais

- Upload de imagem (PNG, JPG, WebP) com remoção automática de fundo
- Geração de modelo 3D com materiais PBR
- Preview 3D interativo no navegador (rotação, zoom, troca de HDRI)
- Download em `.glb` e `.mp4` (turntable)
- Histórico de gerações por usuário
- API REST pública para o plano Studio
- Sistema de créditos com Stripe
- Galeria pública opcional (usuário escolhe se compartilha)

## Considerações de Licença

O TRELLIS.2-4B tem código sob licença MIT, **mas o model card do Hugging Face restringe o uso a pesquisa e fins acadêmicos**. Antes do lançamento comercial:

- Validar termos diretamente com a Microsoft Research
- Avaliar alternativas comerciais (Stability 3D, Tripo, Meshy API) caso necessário
- Versão inicial pode ser lançada como “research preview” gratuita para validar produto

## Estimativa de Custos Mensais (MVP)

- Cloudflare (Workers + R2 + D1 + Pages): ~$5
- Domínio: ~$1
- RunPod Serverless (estimado 5k gerações/mês): ~$30-80
- Stripe: 2.9% + $0.30 por transação
- **Total inicial**: ~$40-100/mês até validar tração

## Roadmap

**Fase 1 - MVP (2-4 semanas)**

- Setup do container TRELLIS.2 no RunPod
- Worker básico com fila e storage
- Frontend simples com upload e preview
- Sem auth, sem pagamento (uso aberto com rate limit por IP)

**Fase 2 - Monetização (2-3 semanas)**

- Auth + sistema de créditos
- Stripe integrado
- Dashboard do usuário com histórico

**Fase 3 - Escala (4+ semanas)**

- API pública com chaves
- Suporte a multi-imagem (várias vistas do mesmo objeto)
- Geração de texturas PBR a partir de mesh existente (segundo modo do TRELLIS.2)
- Galeria pública e comunidade
- Webhooks para integrações (Blender plugin, Unity package)
