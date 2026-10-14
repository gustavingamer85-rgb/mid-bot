# Bot de Middleman (MID) automático — EclipsePay + PurinCash

Bot de Discord em JavaScript (discord.js v14) que intermedeia negociações: o comprador paga via PIX, o valor fica **retido** na conta do gateway e só é enviado ao vendedor quando o comprador libera (ou a staff decide numa disputa).

## Instalação
1. Instale o **Node.js 18.17+** (recomendado 20 ou 22).
2. Na pasta do projeto: `npm install`
3. Copie `.env.example` para `.env` e preencha `DISCORD_TOKEN` e `CLIENT_ID` (`GUILD_ID` e `ADMIN_IDS` são opcionais).
4. `npm start`
5. No Discord Developer Portal, convide o bot com os escopos `bot` + `applications.commands` e as permissões **Gerenciar Canais, Ver Canais, Enviar Mensagens, Anexar Arquivos, Inserir Links, Ler Histórico**.

## Configuração (tudo pelo Discord)
Use **/config** (somente Administradores ou IDs em `ADMIN_IDS`):
- **Chaves EclipsePay / PurinCash**: cola a API key (e o webhook secret, se for usar webhook). O bot testa a chave na hora e mostra o saldo.
- **Gateway ativo**: escolhe qual gateway gera as cobranças.
- **Taxas e limites**: % do MID, taxa mínima, quem paga (comprador/vendedor/dividir), valor mín./máx.
- **Prazos e webhook**: expiração do PIX, cancelamento por inatividade, limite de negociações por usuário, URL pública.
- **Canais e cargos**: categoria dos tickets, canal de logs/transcrições, cargo da staff.
- **Aparência do painel** e **Enviar painel**: publica a mensagem com os botões *Sou o comprador / Sou o vendedor*.

## Fluxo da negociação
1. Alguém clica em *Sou o comprador/vendedor*, escolhe a outra parte e informa valor + descrição (o vendedor também informa a chave PIX). Cria-se um canal privado.
2. A outra parte clica em **Aceitar** (se for o vendedor, informa a chave PIX). O bot gera o PIX (QR Code + copia e cola).
3. Pagamento detectado automaticamente → valor retido. O vendedor entrega.
4. O comprador clica em **Liberar pagamento** → o bot faz o PIX ao vendedor.
5. **Disputa**: a staff chama botões para pagar o vendedor ou reembolsar o comprador (o bot pede a chave PIX ao comprador).
6. Ao finalizar, **Fechar ticket** gera a transcrição no canal de logs.

## Confirmação de pagamento
- Padrão: o bot confere as cobranças pendentes a cada 20 s (não precisa de URL pública).
- Opcional (mais rápido): defina `WEBHOOK_PORT` e exponha por **HTTPS** (EclipsePay exige HTTPS na porta 443; use proxy reverso/Cloudflare Tunnel):
  - EclipsePay: `https://SEU_DOMINIO/webhook/eclipsepay` — cadastre no painel, copie o *secret* para o /config e clique em "Testar" (o bot responde ao desafio).
  - PurinCash: `https://SEU_DOMINIO/webhook/purincash` — o secret fica em Dashboard → Equipe & API → Developer API. Defina a "URL pública" no /config para o bot enviar o `callbackUrl`.
- O webhook só funciona como gatilho: o bot sempre reconfirma o status na API do gateway antes de agir.

## Segurança embutida
- Trava contra saque duplo (estado reservado antes da chamada) e `Idempotency-Key` fixa por tentativa na EclipsePay.
- Falha de rede/5xx num saque → status "incerto": a staff confere o painel do gateway antes de tentar de novo.
- Pagamento que chega depois de a negociação ser cancelada gera alerta para reembolso manual.
- Chaves ficam em `data/config.json` (texto simples): proteja a pasta, não envie ao Git e use permissões restritas no servidor.

## Observações dos gateways
- **EclipsePay**: conta com verificação aprovada; a key precisa das permissões `charges:create/read` e `payouts:create/read`; limite de R$ 1.000 por operação.
- **PurinCash**: saques PIX ficam `pending` até aprovação; ative aprovação automática no painel ou o modo *turbo* (no /config). Limite padrão de 10 saques/hora e valor mínimo de saque por conta (R$ 5 por padrão).
- Taxas de saque dos gateways saem do seu saldo; ajuste a taxa do MID para cobri-las.
- Teste primeiro com chave de sandbox (`ps_test_...` na PurinCash).
