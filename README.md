# 💰 Startup Cashback — Sistema Multi-App (Lojista & Cliente)

Este é um ecossistema de fidelização e engajamento focado em estratégias de **Cashback B2B2C**. O projeto foi estruturado de forma inteligente para rodar de maneira descentralizada através de **dois aplicativos independentes**, mas que conversam e atualizam dados em tempo real sob o mesmo teto (banco de dados).

---

## 🏗️ Arquitetura do Sistema (Multi-App Dependente)

Para o correto funcionamento do ecossistema de cashback, o projeto **depende obrigatoriamente da separação física e lógica em 2 aplicativos distintos**:

1. **Painel do Comerciante (`comerciante.html`):** * **Público-alvo:** Lojistas, operadores de caixa e gerentes de estabelecimentos.
   * **Funções:** Cadastrar novos clientes via CPF, lançar valores de compras, calcular e creditar cashback com porcentagens customizadas, realizar o resgate/desconto de saldos acumulados e criar painéis de ofertas/cupons temporários com prazo de validade.
2. **Carteira do Cliente (`cliente.html`):**
   * **Público-alvo:** Consumidor final (cliente da loja).
   * **Funções:** Consultar o saldo acumulado em tempo real inserindo apenas o CPF, visualizar o histórico (extrato) detalhado de créditos ganhos e resgates efetuados, e receber notificações de ofertas ou cupons de desconto criados sob medida para o seu perfil.

> 💡 **Nota de Produção:** Ambos os arquivos utilizam o mesmo objeto `firebaseConfig`, permitindo que o cliente veja o saldo piscar na tela do smartphone no exato milissegundo em que o caixa confirma a operação no balcão da loja.

---

## 🛠️ Configuração Obrigatória no Firebase Firestore

Como as consultas e transações são indexadas de forma ágil utilizando o **CPF limpo (apenas números)** como o identificador único (`Document ID`) dos clientes, as regras de segurança precisam ser estritas para impedir fraudes ou invasões maliciosas, anulando qualquer brecha ou prazo de expiração (modo de teste).

### Passo a Passo no Console do Firebase:
1. Acesse o [Console do Firebase](https://console.firebase.google.com/) e entre no seu projeto.
2. No menu lateral esquerdo, clique em **Firestore Database**.
3. Acesse a aba **Regras (Rules)** no topo da página.
4. Apague todo o conteúdo existente e cole o código de segurança definitivo abaixo:

```javascript
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {
    
    // --- COLEÇÃO DE USUÁRIOS E SALDOS ---
    match /usuarios/{cpf} {
      // Permite que o caixa e o cliente consultem e iniciem cadastros pelo CPF
      allow read, create: if true;
      
      // CRÍTICO: Bloqueia 100% qualquer tentativa de DELETAR um usuário do sistema
      allow delete: if false;
      
      // Validação estrita para impedir que campos inconsistentes modifiquem o saldo
      allow update: if request.resource.data.cpf == cpf 
                    && request.resource.data.saldo_cashback is number;
    }
    
    // --- COLEÇÃO DE HISTÓRICO DE COMPRAS (EXTRATO) ---
    match /historico_compras/{docId} {
      // Permite que o caixa registre a movimentação e o cliente puxe o extrato
      allow read, create: if true;
      
      // CRÍTICO: Uma vez gravada a transação (ganho ou resgate), ela se torna IMUTÁVEL
      allow update, delete: if false;
    }
    
    // --- COLEÇÃO DE OFERTAS E CUPONS ---
    match /ofertas/{docId} {
      // Permite lançar novas campanhas e o cliente visualizá-las no app
      allow read, create: if true;
      
      // Bloqueia a alteração ou exclusão de cupons ativos por terceiros
      allow update, delete: if false;
    }
  }
}
