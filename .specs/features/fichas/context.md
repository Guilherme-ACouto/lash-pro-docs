# Fichas — Decisões do Usuário

## Quem preenche a anamnese
**Decisão: Os dois modos**

A profissional pode preencher diretamente no sistema OU enviar um link público para a cliente preencher sozinha no celular.

**Implicações:**
- Backend precisa de endpoint público (sem JWT) com acesso por token único
- Fluxo: profissional gera token → copia link → envia por WhatsApp → cliente acessa URL pública e preenche
- Página pública de anamnese: sem sidebar/bottom nav, layout limpo estilo formulário
- Token deve expirar (sugestão: 7 dias) e ser invalidado após uso

## Mapping vinculado ao agendamento
**Decisão: Independente (sem vínculo)**

Ficha de mapping criada livremente, vinculada apenas ao cliente e à data. Não requer agendamento existente.

**Implicações:**
- Tabela `lash_mappings` com `client_id` + `created_at` — sem FK para appointments
- Histórico cronológico simples

## Canvas de desenho
**Decisão: Traço livre + cores + espessura**

Profissional pode desenhar com lápis freehand e escolher:
- Cor do traço (ex: diferentes cores para olho esquerdo/direito ou tipos de extensão)
- Espessura do traço

**Implicações:**
- HTML5 Canvas com API de drawing events (mouse + touch)
- Seletor de cor (color picker compacto: 6-8 cores predefinidas + custom)
- Seletor de espessura (3 opções: fina / média / grossa)
- Canvas exportado como base64 PNG para persistência

## Armazenamento das fotos
**Decisão: Base64 no banco de dados**

Fotos antes/depois armazenadas como TEXT base64 direto na tabela `lash_mappings`.

**Implicações:**
- Colunas: `photo_before TEXT`, `photo_after TEXT`
- Frontend comprime imagem antes de enviar (max ~800px, qualidade 0.7) para manter tamanho controlado
- Sem endpoint de upload separado — foto vai junto com o JSON da ficha
