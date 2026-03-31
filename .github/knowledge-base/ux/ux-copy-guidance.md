# UX Copy Guidance

**Phase 1 Deliverable — Checkout UX & Flow Design**
**Profile**: Hosted Checkout | **Payment Methods**: credit_card, pix, boleto | **Provider**: Stripe | **Currency**: BRL

---

## 1. Guiding Principles for Payment Copy

- **Be direct**: Tell users exactly what happened and what to do next. Avoid vague phrases like "something went wrong."
- **Be calm**: Avoid alarming language ("error!", "failed!") for recoverable situations. Use neutral, informative tone.
- **Be actionable**: Every error message must have a clear next step.
- **Never expose technical details**: Do not surface Stripe error codes, HTTP status codes, or internal identifiers to users.
- **Localization note**: All copy defaults to Brazilian Portuguese (pt-BR). English is shown as reference only.

---

## 2. Step Labels and CTAs

| Location | Copy (pt-BR) | Copy (en reference) |
|----------|-------------|---------------------|
| Checkout entry button | "Ir para o pagamento" | "Proceed to Checkout" |
| Continue to payment CTA | "Continuar para pagamento" | "Continue to Payment" |
| Submit / Place order CTA | "Confirmar pedido" | "Place Order" |
| Submit loading state | "Processando…" | "Processing…" |
| Pix submit CTA | "Gerar QR Code" | "Generate QR Code" |
| Boleto submit CTA | "Emitir boleto" | "Generate Boleto" |
| Retry button | "Tentar novamente" | "Try Again" |
| Cancel / Return to cart | "Voltar ao carrinho" | "Return to Cart" |
| Edit section link | "Editar" | "Edit" |
| Copy code button | "Copiar código" | "Copy Code" |
| Copy barcode button | "Copiar código de barras" | "Copy Barcode" |
| Download boleto | "Baixar boleto PDF" | "Download Boleto PDF" |
| "I have paid" (Pix) | "Já efetuei o pagamento" | "I've Completed Payment" |

---

## 3. Field-Level Validation Messages

| Field | Error (pt-BR) | Error (en reference) |
|-------|--------------|----------------------|
| Full Name | "Por favor, insira seu nome completo" | "Please enter your full name" |
| Email | "Insira um endereço de e-mail válido" | "Please enter a valid email address" |
| Phone | "Insira um número de telefone válido" | "Please enter a valid phone number" |
| CEP | "CEP inválido — verifique e tente novamente" | "Invalid postal code — please check and try again" |
| CEP not found | "CEP não encontrado. Preencha o endereço manualmente." | "Postal code not found. Please fill in the address manually." |
| Street | "Informe o nome da rua" | "Please enter the street name" |
| Address number | "Informe o número do endereço" | "Please enter the address number" |
| City | "Informe a cidade" | "Please enter the city" |
| State | "Selecione um estado" | "Please select a state" |
| Cardholder Name | "Insira o nome como aparece no cartão" | "Please enter the name as shown on the card" |
| Card Number | "Número do cartão inválido" | "Invalid card number" |
| Expiry Date | "Data de validade inválida" | "Invalid expiry date" |
| CVV | "CVV inválido" | "Invalid CVV" |
| CPF | "CPF inválido — verifique os números" | "Invalid CPF — please check the number" |
| CNPJ | "CNPJ inválido — verifique os números" | "Invalid CNPJ — please check the number" |
| Promo Code (invalid) | "Código inválido ou expirado" | "Invalid or expired code" |
| Promo Code (applied) | "Desconto aplicado: {value}" | "Discount applied: {value}" |
| Required field (generic) | "Este campo é obrigatório" | "This field is required" |

---

## 4. Payment Decline Messages

Map Stripe decline codes to user-facing copy. Never expose raw decline codes.

| Stripe Decline Code | User-Facing Message (pt-BR) | User-Facing Message (en reference) | Recovery CTA |
|---------------------|----------------------------|------------------------------------|--------------|
| `card_declined` (generic) | "Seu cartão foi recusado. Verifique os dados ou tente outro cartão." | "Your card was declined. Check the details or try a different card." | "Tentar outro cartão" |
| `insufficient_funds` | "Saldo insuficiente. Tente outro cartão ou forma de pagamento." | "Insufficient funds. Try a different card or payment method." | "Escolher outra forma" |
| `expired_card` | "Seu cartão está vencido. Use um cartão com data válida." | "Your card has expired. Please use a valid card." | "Usar outro cartão" |
| `incorrect_cvc` | "CVV incorreto. Verifique o código no verso do cartão." | "Incorrect CVV. Check the code on the back of your card." | "Corrigir CVV" |
| `incorrect_number` | "Número do cartão incorreto. Verifique e tente novamente." | "Incorrect card number. Please check and try again." | "Corrigir número" |
| `do_not_honor` | "Seu banco recusou a transação. Entre em contato com seu banco ou use outro cartão." | "Your bank declined the transaction. Contact your bank or use a different card." | "Tentar outro cartão" |
| `fraudulent` / `pickup_card` | "Não foi possível processar este cartão. Use outro cartão ou forma de pagamento." | "We couldn't process this card. Please use a different card or payment method." | "Escolher outra forma" |
| `processing_error` | "Erro ao processar. Aguarde um momento e tente novamente." | "Processing error. Please wait a moment and try again." | "Tentar novamente" |
| Generic / unknown | "Não foi possível completar o pagamento. Tente novamente ou escolha outra forma." | "We couldn't complete the payment. Try again or choose another method." | "Tentar novamente" |

---

## 5. System / Network Error Messages

| Situation | User-Facing Message (pt-BR) | User-Facing Message (en reference) |
|-----------|-----------------------------|-------------------------------------|
| Network timeout | "Conexão interrompida. Verifique sua internet e tente novamente." | "Connection interrupted. Check your internet and try again." |
| Server error (5xx) | "Estamos com uma instabilidade. Aguarde um instante e tente novamente." | "We're experiencing an issue. Please wait a moment and try again." |
| Duplicate order detected | "Seu pedido já foi recebido. Redirecionando para a confirmação…" | "Your order was already received. Redirecting to your confirmation…" |
| Session expired | "Sua sessão expirou. Faça login novamente — seu carrinho foi salvo." | "Your session has expired. Please log in again — your cart has been saved." |
| Payment method unavailable | "Esta forma de pagamento não está disponível no momento. Tente outra." | "This payment method is currently unavailable. Please try another." |

---

## 6. Pix-Specific Copy

| Moment | Copy (pt-BR) | Copy (en reference) |
|--------|-------------|---------------------|
| QR Modal title | "Pague com Pix" | "Pay with Pix" |
| Timer running | "Seu código expira em {mm:ss}" | "Your code expires in {mm:ss}" |
| Timer expired | "Código expirado. Gere um novo para continuar." | "Code expired. Generate a new one to continue." |
| Regenerate button | "Gerar novo código Pix" | "Generate New Pix Code" |
| Payment detected | "Pagamento confirmado! Redirecionando…" | "Payment confirmed! Redirecting…" |
| Dismiss warning | "Ainda não recebemos seu pagamento. Tem certeza que deseja sair?" | "We haven't received your payment yet. Are you sure you want to leave?" |

---

## 7. Boleto-Specific Copy

| Moment | Copy (pt-BR) | Copy (en reference) |
|--------|-------------|---------------------|
| Boleto issued title | "Boleto gerado" | "Boleto Generated" |
| Expiry notice | "Pague até {date} para garantir seu pedido." | "Pay by {date} to secure your order." |
| Email notice | "Uma cópia foi enviada para {email}" | "A copy was sent to {email}" |
| Awaiting payment status | "Aguardando pagamento" | "Awaiting Payment" |

---

## 8. Confirmation Page Copy

| Element | Copy (pt-BR) | Copy (en reference) |
|---------|-------------|---------------------|
| Page title | "Pedido confirmado!" | "Order Confirmed!" |
| Order number label | "Número do pedido:" | "Order number:" |
| Email confirmation note | "Enviamos a confirmação para {email}" | "We sent a confirmation to {email}" |
| Continue shopping CTA | "Continuar comprando" | "Continue Shopping" |
| Card charge line | "Cartão terminando em {last4}" | "Card ending in {last4}" |
| Pix pending note | "Pagamento Pix registrado e sendo processado." | "Pix payment recorded and being processed." |
| Boleto pending note | "Aguardamos o pagamento do boleto para confirmar o pedido." | "We're waiting for boleto payment to confirm your order." |
