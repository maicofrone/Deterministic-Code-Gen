# Deterministic Code Gen — Technical Documentation

**AI-assisted project disclosure:** This project and its documentation were created with the assistance of Artificial Intelligence (AI), specifically ChatGPT, with the project being developed by **maicofrone**. AI assistance was used to help design, explain, structure, and implement the software. The code itself should be reviewed, tested, and audited by a human before being relied upon for sensitive credentials.

**Declaração sobre o uso de IA no projeto:** Este projeto e sua documentação foram criados com o auxílio de Inteligência Artificial (IA), especificamente o ChatGPT, com o projeto sendo desenvolvido por **maicofrone**. A IA foi utilizada para auxiliar no desenvolvimento, na explicação, na estruturação e na implementação do software. O próprio código deve ser revisado, testado e auditado por um ser humano antes de ser utilizado para proteger credenciais sensíveis.

---

# English

## 1. Overview

This project is a **local, deterministic code generator based on HMAC-SHA-256**.

Its intended concept is to act as a **second, final secret layer on top of a password manager** such as **1Password, Bitwarden, or Proton Pass**.

The idea is:

1. The password manager stores the normal website/application password.
2. The password manager autofills that password.
3. The user enters the **name of the website or application being accessed** into this generator. Using the site/app name as the name is recommended because it makes the generated code different for each service.
4. The generator uses the independent password and PIN to deterministically calculate an additional code.
5. The user enters that generated code as the final secret required by the target service, when the target service supports an additional password/PIN/code field.
6. The generator does not save the password or PIN, and the generated code does not need to be stored in the password manager.

This creates a practical separation between the credential stored in the password manager and an additional secret that must be reproduced independently.

**Important security qualification:** the current implementation is **not TOTP and is not a standards-based two-factor authentication (2FA) authenticator**. The original application explicitly identifies itself as a deterministic HMAC-SHA-256 generator rather than TOTP or 2FA. fileciteturn1file0L76-L114

Therefore, this project should be described as a **deterministic second-secret layer / password-companion mechanism**, not as replacement for hardware security keys, passkeys, TOTP, or another formal MFA mechanism.

---

## 2. Core Security Concept

The key idea is that the password manager does **not** need to contain every secret required to access an account.

For example:

```text
Password Manager
      │
      └── Stored password
              │
              ▼
       Website login
              │
              ▼
     Additional secret/code
              ▲
              │
     This application
```

A password manager such as 1Password, Bitwarden, or Proton Pass can therefore remain responsible for storing and autofilling the primary password, while this application is responsible for reproducing an additional deterministic secret.

The project does not send the entered name, password, PIN, or generated code to a server. The source states that processing is performed locally in the browser, without a database, external API, or third-party service, and that the password and PIN are not stored in `localStorage` or `sessionStorage`.

The **password and PIN are intentionally not saved by the application**. This is a central part of the security concept: if an attacker compromises the computer, browser environment, or another place where the application data might otherwise have been persisted, the application should not provide a stored copy of the independent password and PIN that could be combined with the password-manager vault.

The intended security model is therefore based on **separation of secrets**:

```text
Password Manager
    └── Primary account passwords

This Generator
    ├── Does NOT save the independent password
    └── Does NOT save the PIN

Result
    └── Deterministic code generated when needed
```

This does not protect against a compromised device that captures secrets while they are being typed. It specifically means that the application itself is not intended to maintain a stored database containing the password and PIN. fileciteturn1file1L194-L237

---

## 3. What the User Enters

The generator requires three inputs:

- **Name**
- **Password**
- **4-digit PIN**

All three are required. The code has no default password or default PIN. fileciteturn1file0L117-L161

Conceptually:

```text
Name + Password + PIN
          │
          ▼
   Deterministic algorithm
          │
          ▼
   12-character code
```

The important property is that the same inputs always produce the same output.

---

## 4. Recommended Use of the Name

The **Name** field should normally contain the name of the website or application for which the user is generating the additional secret.

Examples:

```text
Google
Facebook
Steam
GitHub
Bank App
Company ERP
```

This is important because the name participates directly in the calculation. Therefore, changing the name changes the resulting code.

For example:

```text
Name = Google
Password = same secret
PIN = same PIN
        ↓
Code A

Name = Facebook
Password = same secret
PIN = same PIN
        ↓
Code B
```

The recommended practice is therefore:

> **Use the name of the website or application being accessed as the Name field.**

The same name should be used consistently for the same service. If the name is changed later, the generated code will also change.

This gives the system a useful separation between services: even when the same independent password and PIN are used, different site/application names produce different deterministic codes.

## 4. Name Normalization

Before the name participates in the algorithm, it is normalized.

The implementation:

- removes leading and trailing whitespace;
- converts the name to lowercase using the `pt-BR` locale;
- applies Unicode NFC normalization.

The implementation is:

```javascript
return valor
  .trim()
  .toLocaleLowerCase("pt-BR")
  .normalize("NFC");
```

This means that superficial differences in capitalization or surrounding spaces do not unintentionally create different results. fileciteturn1file6L1083-L1096

Example:

```text
Input:
    Gerador

Normalized:
    gerador
```

---

## 5. Counting Letters, Vowels and Consonants

The normalized name is scanned character by character.

Only Unicode letters are counted.

The application calculates:

```text
letters
vowels
consonants
```

The implementation uses Unicode property matching (`\p{L}`) and a predefined set of vowels, including accented Portuguese vowels. fileciteturn1file6L1099-L1156

Example:

```text
gerador

Letters:       7
Vowels:        3
Consonants:    4
```

---

## 6. PIN Interpretation

The PIN must contain exactly four numeric digits.

It is interpreted as:

```text
PIN = A B C D

AB = first two digits
C  = third digit
D  = fourth digit
CD = last two digits
```

For:

```text
PIN = 0123
```

the values are:

```text
AB = 01 → 1
C  = 2
D  = 3
CD = 23
```

This interpretation is implemented directly by `interpretarPin()`. fileciteturn1file6L1159-L1191

---

## 7. BASE Calculation

The PIN-derived values and the name statistics are combined into a deterministic numerical value called `BASE`.

The implemented formula is:

```text
BASE =
    (letters × AB)
  + (vowels × C)
  + (consonants × D)
  + (AB × CD)
```

For the example:

```text
letters    = 7
vowels     = 3
consonants = 4

AB = 1
C  = 2
D  = 3
CD = 23
```

the calculation is:

```text
7 × 1 = 7
3 × 2 = 6
4 × 3 = 12
1 × 23 = 23

7 + 6 + 12 + 23 = 48

BASE = 48
```

This formula is implemented by `calcularBase()`. fileciteturn1file5L913-L974

---

## 8. Creation of the HMAC Message

The normalized name, original four-digit PIN, and BASE are combined into a fixed message:

```text
name=NORMALIZED_NAME|pin=PIN|base=BASE
```

Example:

```text
name=gerador|pin=0123|base=48
```

Because the same inputs produce the same normalized name, BASE, and message, this stage is deterministic. fileciteturn1file4L738-L767

---

## 9. Password as the HMAC Key

The password is not simply appended to the message.

Instead, the password becomes the **cryptographic key** used by HMAC-SHA-256.

Conceptually:

```text
HMAC key:
    Password

Message:
    name=gerador|pin=0123|base=48
```

The implementation uses the browser's native **Web Crypto API**, specifically `crypto.subtle`, to import the password as an HMAC key and perform the operation. fileciteturn1file4L772-L800

---

## 10. HMAC-SHA-256

The browser calculates:

```text
HMAC-SHA-256(
    key = password,
    message = name + PIN + BASE
)
```

The result is:

```text
256 bits
=
32 bytes
```

The code performs this operation using the native Web Crypto API rather than implementing SHA-256 manually. fileciteturn1file6L1249-L1303

The high-level flow is:

```text
Password
   +
Deterministic message
   ↓
HMAC-SHA-256
   ↓
32-byte result
```

---

## 11. Conversion to BigInt

The 32 HMAC bytes are interpreted as one positive JavaScript `BigInt`.

This is necessary because a normal JavaScript `Number` cannot safely represent the entire 256-bit HMAC value.

The implementation shifts the current number by 8 bits and incorporates each byte:

```text
32 bytes
   ↓
BigInt
```

The source explicitly uses `BigInt` to avoid the precision limitations of conventional JavaScript numbers. fileciteturn1file7L1372-L1397

---

## 12. Reduction to 12 Base62 Characters

The resulting integer is reduced with:

```text
number % 62^12
```

This restricts the numerical result to the range representable by exactly 12 Base62 positions. fileciteturn1file7L1402-L1427

---

## 13. Base62 Encoding

The application uses this 62-character alphabet:

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZ
abcdefghijklmnopqrstuvwxyz
0123456789
```

That gives:

```text
26 uppercase letters
+ 26 lowercase letters
+ 10 digits
= 62 symbols
```

The resulting number is repeatedly divided by 62, with each remainder becoming a Base62 character. fileciteturn1file2L378-L395

The final result is exactly:

```text
12 characters
```

fileciteturn1file2L400-L423

---

## 14. Complete Algorithm

The complete algorithm can be represented as:

```text
Name
  ↓
Trim + lowercase + Unicode NFC
  ↓
Count letters / vowels / consonants
  ↓
PIN
  ↓
Interpret AB / C / D / CD
  ↓
Calculate BASE
  ↓
Create deterministic message
  ↓
Password as HMAC key
  ↓
HMAC-SHA-256
  ↓
32 bytes
  ↓
BigInt
  ↓
Modulo 62^12
  ↓
Base62 conversion
  ↓
12-character deterministic code
```

This is the algorithm implemented by the application. fileciteturn1file0L90-L114

---

## 15. Deterministic Behavior

The generator does not use:

- random numbers;
- current time;
- dates;
- TOTP counters.

Therefore:

```text
Same Name
+
Same Password
+
Same PIN
=
Same Code
```

Changing any of the inputs changes the expected result. fileciteturn1file2L436-L485

This property is fundamental to the intended password-manager companion use case: the user does not need to save the generated code itself.

---

## 16. Why This Can Be Used Alongside a Password Manager

Suppose a website account has:

```text
Username
Password
Additional secret/code
```

A possible workflow is:

```text
1. Password manager provides the username.
2. Password manager provides the primary password.
3. User opens the local generator.
4. User provides the independent secret inputs.
5. Generator reproduces the deterministic 12-character code.
6. User enters the generated code into the additional field.
```

The password manager therefore does not have to contain the final secret.

This can reduce the consequences of a situation where someone obtains access to the password-manager vault: possession of the vault alone does not automatically reveal the separate inputs required to reproduce the companion code.

**However:** this is a security design concept, not a guarantee. The actual protection depends on how the separate inputs are managed, whether the device/browser is compromised, whether malware or keyloggers are present, and whether the target service really requires the additional secret.

---

## 17. Important Distinction: This Is Not Formal 2FA

This distinction is critical.

The application itself states:

> It is not TOTP and should not be presented as a two-factor authentication system.

fileciteturn1file1L242-L268

Why?

Traditional 2FA generally combines independent authentication factors, such as:

```text
Something you know
+
Something you have
```

or:

```text
Password
+
Hardware security key
```

or:

```text
Password
+
TOTP authenticator
```

This application instead produces another deterministic secret from secret inputs.

Therefore, the most technically accurate description is:

> **A deterministic second-secret layer used in addition to credentials stored in a password manager.**

It should **not** be marketed or documented as a standards-compliant MFA/2FA authenticator.

---

## 18. Local Processing and Privacy

According to the implementation, the generator operates entirely inside the browser.

It does not send:

- Name
- Password
- PIN
- Generated code

to a server.

The project does not use:

- a database;
- an external API;
- a third-party backend;
- `localStorage`;
- `sessionStorage`;
- `Math.random()`;
- time/date values;
- TOTP.

The HMAC operation uses the browser's Web Crypto API. fileciteturn1file3L661-L703

This makes the architecture simple:

```text
User
  ↓
Browser
  ↓
JavaScript
  ↓
Web Crypto API
  ↓
Result
```

No server is required by the current implementation.

---

## 19. Automatic Updating

The generator watches the three input fields.

When the:

- Name changes;
- Password changes;
- PIN changes;

a new generation is requested.

fileciteturn1file3L580-L612

The UI therefore does not require a separate "Generate" button for every modification.

---

## 20. Protection Against Old Asynchronous Results

HMAC generation is asynchronous.

If the user changes the inputs rapidly, several calculations could theoretically be running at the same time.

The application solves this with a generation identifier.

Conceptually:

```text
Generation 1 starts
      ↓
User changes data
      ↓
Generation 2 starts
      ↓
Generation 2 finishes
      ↓
Generation 1 finishes later
      ↓
Generation 1 result is ignored
```

Only the most recent generation is allowed to update the displayed code. fileciteturn1file3L617-L656

---

## 21. User Interface Features

The application also includes:

- dark/light theme;
- password visibility toggle;
- PIN visibility toggle;
- PIN numeric validation;
- generated-code display;
- copy-to-clipboard functionality;
- clear/reset functionality;
- an integrated "How it works?" explanation;
- responsive layout for smaller screens;
- reduced-motion support.

The UI presents the generator and a separate explanation page. fileciteturn1file8L1537-L1647

---

## 22. Example

Using:

```text
Name:
Gerador

Password:
senha123

PIN:
0123
```

the process becomes:

```text
Gerador
   ↓
gerador
   ↓
7 letters
3 vowels
4 consonants
   ↓
PIN 0123
   ↓
01 | 2 | 3
   ↓
BASE = 48
   ↓
name=gerador|pin=0123|base=48
   ↓
senha123 as HMAC key
   ↓
HMAC-SHA-256
   ↓
BigInt
   ↓
modulo 62^12
   ↓
Base62
   ↓
12-character code
```

This complete flow is also represented in the application's own documentation. fileciteturn1file9L1658-L1708

---

## 23. Recommended Intended Architecture

For the intended password-manager companion concept:

```text
                    ┌─────────────────────┐
                    │   Password Manager  │
                    │                     │
                    │ Username            │
                    │ Primary Password    │
                    └──────────┬──────────┘
                               │
                               ▼
                       Primary login
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Target Application  │
                    │ / Website           │
                    └──────────┬──────────┘
                               │
                     Additional secret
                               ▲
                               │
                    ┌─────────────────────┐
                    │ Deterministic       │
                    │ Code Generator      │
                    │                     │
                    │ Name + Password +   │
                    │ PIN                 │
                    └─────────────────────┘
```

The important architectural principle is **separation of secrets**.

The primary credential may live in the password manager, while the additional deterministic secret is derived independently.

---

## 24. Security Limitations

This project should not be treated as a replacement for professional authentication systems.

Important limitations include:

1. **The generated code is deterministic.** It does not automatically rotate with time.
2. **It is not TOTP.**
3. **It is not a hardware security key.**
4. **It is not a passkey/WebAuthn credential.**
5. **The PIN is only four digits**, so it should not be treated as a strong secret by itself.
6. The security of the system depends heavily on the secrecy and quality of the password and other inputs.
7. A compromised device or browser can potentially expose secrets entered into the application.
8. A keylogger or malicious browser extension could capture input.
9. The target website must actually support an additional secret field for this workflow to be useful.
10. The project has not, by itself, established resistance against all password-cracking, phishing, malware, browser, or endpoint attacks.

The source itself warns that the project does not replace professional authentication, credential management, or security systems designed specifically for those purposes. fileciteturn1file1L242-L268

---

## 25. AI-Assisted Development and Documentation Disclosure

This project was created with assistance from **Artificial Intelligence (AI)**.

The use of AI applies not only to parts of the source code, but also to the documentation itself. **This `.md` documentation file was also created with the assistance of AI (ChatGPT).**

In practical terms:

```text
Project / code:
AI-assisted

Technical explanations:
AI-assisted

Documentation:
AI-assisted

This .md file:
AI-assisted
```

Project information:

```text
Project author:
maicofrone

AI assistant:
ChatGPT
```

AI was used to assist with software design, implementation, explanation, organization, revision, and documentation.

This disclosure is intentional and should remain part of the project's documentation. The fact that AI was used does not constitute a security certification. AI-assisted code should be reviewed and tested by humans, especially when the software is intended to protect authentication secrets.

 fileciteturn0file0L1187-L1194

---

# Português

## 1. Visão geral

Este projeto é um **gerador determinístico de códigos baseado em HMAC-SHA-256**.

A ideia de utilização é funcionar como uma **segunda camada secreta e final sobre um gerenciador de senhas**, como **1Password, Bitwarden ou Proton Pass**.

O conceito é:

1. O gerenciador de senhas armazena a senha normal do serviço.
2. O gerenciador preenche automaticamente essa senha.
3. O usuário informa neste gerador o **nome do site ou aplicativo que está acessando**. Recomenda-se usar o nome do próprio site/app, pois isso faz com que cada serviço produza um código diferente.
4. O gerador utiliza a senha e o PIN independentes para calcular deterministicamente um código adicional.
5. O usuário informa esse código como segredo adicional, quando o serviço possui um campo adicional de senha/PIN/código.
6. O gerador não salva a senha nem o PIN, e o código gerado não precisa ser armazenado no gerenciador de senhas.

Assim, o gerenciador continua responsável pela senha principal, enquanto este projeto fornece uma camada adicional que precisa ser reproduzida separadamente.

**Importante:** tecnicamente, o projeto atual **não é TOTP e não é um autenticador 2FA padronizado**. O próprio código deixa isso explícito. fileciteturn1file0L76-L114

A descrição tecnicamente correta é:

> **Camada determinística de segundo segredo para uso complementar a um gerenciador de senhas.**

Não deve ser apresentado como substituto de passkeys, chaves físicas, TOTP ou outros mecanismos formais de MFA.

---

## 2. Conceito de segurança

O objetivo principal é evitar que o gerenciador de senhas precise armazenar **todos** os segredos necessários para acessar uma conta.

Exemplo:

```text
Gerenciador de senhas
        │
        └── Senha principal
                │
                ▼
          Login principal
                │
                ▼
       Código/segredo adicional
                ▲
                │
        Este aplicativo
```

Assim, 1Password, Bitwarden ou Proton Pass podem continuar armazenando a senha principal, enquanto este aplicativo é usado para reproduzir um segundo segredo determinístico.

O código atual processa os dados localmente no navegador e, segundo sua implementação, não envia nome, senha, PIN ou código para um servidor. Também não utiliza banco de dados, API externa ou serviço de terceiros, e não armazena senha/PIN em `localStorage` ou `sessionStorage`.

A **senha e o PIN não são salvos pela aplicação de propósito**. Isso é uma parte central do conceito de segurança: se alguém conseguir comprometer o computador, o navegador ou outro ambiente em que os dados poderiam ser persistidos, a própria aplicação não deve fornecer uma cópia armazenada da senha independente e do PIN que possa ser combinada com o conteúdo do gerenciador de senhas.

O modelo pretendido é, portanto, baseado na **separação dos segredos**:

```text
Gerenciador de senhas
    └── Senhas principais das contas

Este gerador
    ├── NÃO salva a senha independente
    └── NÃO salva o PIN

Resultado
    └── Código determinístico gerado quando necessário
```

Isso não protege contra um dispositivo comprometido que capture os segredos enquanto eles estão sendo digitados. O objetivo específico é que a própria aplicação não mantenha uma base armazenada contendo a senha e o PIN. fileciteturn1file1L194-L237

---

## 3. Entradas

O sistema recebe três informações:

- **Nome**
- **Senha**
- **PIN de 4 dígitos**

Todas são obrigatórias. fileciteturn1file0L117-L161

Fluxo:

```text
Nome + Senha + PIN
        ↓
Algoritmo determinístico
        ↓
Código de 12 caracteres
```

A propriedade fundamental é:

```text
mesmas entradas
      ↓
mesmo código
```

---

## 4. Como escolher o nome — recomendação principal

O campo **Nome** deve, de preferência, receber o **nome do site ou aplicativo no qual o usuário está fazendo login**.

Exemplos:

```text
Google
Facebook
Steam
GitHub
Aplicativo do banco
ERP da empresa
```

Isso é importante porque o nome participa diretamente do cálculo do código. Portanto, **alterar o nome altera o código final**.

Exemplo:

```text
Nome = Google
Senha = mesma senha independente
PIN = mesmo PIN
        ↓
Código A

Nome = Facebook
Senha = mesma senha independente
PIN = mesmo PIN
        ↓
Código B
```

A recomendação de uso é:

> **Use como Nome o próprio site ou aplicativo que está sendo acessado.**

Para o mesmo serviço, o usuário deve manter sempre o mesmo nome. Se o nome for alterado posteriormente, o código também será alterado.

Isso cria uma separação útil entre os serviços: mesmo que a mesma senha independente e o mesmo PIN sejam utilizados, sites/aplicativos diferentes produzirão códigos determinísticos diferentes.

## 4. Normalização do nome

O nome passa por três etapas:

1. remoção de espaços no começo e no final;
2. conversão para letras minúsculas;
3. normalização Unicode NFC.

Implementação:

```javascript
return valor
  .trim()
  .toLocaleLowerCase("pt-BR")
  .normalize("NFC");
```

fileciteturn1file6L1083-L1096

Exemplo:

```text
Gerador
   ↓
gerador
```

---

## 5. Contagem das letras

Depois da normalização, o sistema percorre o nome e calcula:

```text
quantidade de letras
quantidade de vogais
quantidade de consoantes
```

Somente caracteres reconhecidos como letras Unicode entram na contagem. O código também considera vogais acentuadas do português. fileciteturn1file6L1099-L1156

Para:

```text
gerador
```

temos:

```text
7 letras
3 vogais
4 consoantes
```

---

## 6. Interpretação do PIN

O PIN precisa ter exatamente quatro números.

Ele é dividido assim:

```text
PIN = A B C D

AB = primeiros dois dígitos
C  = terceiro
D  = quarto
CD = últimos dois dígitos
```

Exemplo:

```text
PIN = 0123

AB = 01 → 1
C  = 2
D  = 3
CD = 23
```

Essa operação é realizada pela função `interpretarPin()`. fileciteturn1file6L1159-L1191

---

## 7. Cálculo da BASE

Depois, o sistema combina as estatísticas do nome com o PIN.

A fórmula é:

```text
BASE =
    (letras × AB)
  + (vogais × C)
  + (consoantes × D)
  + (AB × CD)
```

Exemplo:

```text
letras    = 7
vogais    = 3
consoantes = 4

AB = 1
C  = 2
D  = 3
CD = 23
```

Resultado:

```text
7 × 1 = 7
3 × 2 = 6
4 × 3 = 12
1 × 23 = 23

7 + 6 + 12 + 23 = 48

BASE = 48
```

fileciteturn1file5L913-L974

---

## 8. Criação da mensagem

O sistema cria uma mensagem padronizada:

```text
name=NOME_NORMALIZADO|pin=PIN|base=BASE
```

Exemplo:

```text
name=gerador|pin=0123|base=48
```

Essa mensagem é determinística. fileciteturn1file4L738-L767

---

## 9. A senha como chave criptográfica

A senha não é simplesmente concatenada com a mensagem.

Ela é utilizada como **chave do HMAC**.

```text
Chave:
senha

Mensagem:
name=gerador|pin=0123|base=48
```

O navegador utiliza a Web Crypto API através de `crypto.subtle`. fileciteturn1file4L772-L800

---

## 10. HMAC-SHA-256

O processo então é:

```text
Senha
   +
Mensagem determinística
   ↓
HMAC-SHA-256
   ↓
32 bytes
```

O HMAC-SHA-256 produz 256 bits, equivalentes a 32 bytes. fileciteturn1file7L1352-L1367

A implementação utiliza a API criptográfica nativa do navegador. fileciteturn1file6L1249-L1303

---

## 11. Conversão para BigInt

Os 32 bytes do HMAC são interpretados como um único número inteiro positivo usando `BigInt`.

Isso é importante porque um `Number` comum do JavaScript não possui precisão suficiente para representar integralmente um valor de 256 bits.

```text
32 bytes
   ↓
BigInt
```

fileciteturn1file7L1372-L1397

---

## 12. Módulo 62¹²

Depois:

```text
BigInt % 62¹²
```

Isso limita o resultado ao espaço necessário para gerar exatamente 12 caracteres Base62. fileciteturn1file7L1402-L1427

---

## 13. Base62

O alfabeto utilizado é:

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZ
abcdefghijklmnopqrstuvwxyz
0123456789
```

São:

```text
26 maiúsculas
26 minúsculas
10 números

Total = 62 caracteres
```

fileciteturn1file2L378-L395

O resultado final possui:

```text
12 caracteres
```

fileciteturn1file2L400-L423

---

## 14. Fluxo completo

```text
Nome
 ↓
Normalização
 ↓
Contagem de letras/vogais/consoantes
 ↓
PIN
 ↓
Cálculo da BASE
 ↓
Mensagem
 ↓
Senha como chave HMAC
 ↓
HMAC-SHA-256
 ↓
BigInt
 ↓
Módulo 62¹²
 ↓
Base62
 ↓
Código final de 12 caracteres
```

fileciteturn1file0L90-L114

---

## 15. Por que o código é determinístico?

O sistema não utiliza:

- números aleatórios;
- horário;
- data;
- contador TOTP.

Portanto:

```text
Mesmo nome
+
Mesma senha
+
Mesmo PIN
=
Mesmo código
```

Se qualquer entrada mudar, o resultado esperado também muda. fileciteturn1file2L436-L485

Essa é justamente a característica que permite usar o projeto como um segredo complementar sem precisar armazenar o código final.

---

## 16. Utilização junto com 1Password, Bitwarden ou Proton Pass

Um cenário de uso seria:

```text
              GERENCIADOR
             DE SENHAS
                  │
        ┌─────────┴─────────┐
        │                   │
     Usuário              Senha
        │                   │
        └─────────┬─────────┘
                  ▼
             Login
                  │
                  ▼
        Campo adicional
                  ▲
                  │
          Código gerado
                  │
                  ▲
        ┌─────────┴─────────┐
        │                   │
      Nome              PIN + senha
        │                   │
        └──── Gerador ──────┘
```

A proposta é manter uma **separação entre os segredos**.

O gerenciador pode armazenar a senha principal, enquanto o segredo adicional é reproduzido pelo gerador.

---

## 17. Por que isso pode ser interessante?

Imagine que alguém consiga acesso ao seu cofre do gerenciador de senhas.

Se o cofre contém somente:

```text
Usuário
Senha principal
```

e o segredo adicional necessário para determinada conta não está armazenado nele, o invasor ainda não possui automaticamente todos os dados usados pelo gerador.

Isso cria uma **camada adicional de separação**.

Entretanto, isso não significa segurança absoluta.

Um dispositivo comprometido, malware, keylogger, extensão maliciosa do navegador ou outro mecanismo capaz de observar o que é digitado pode comprometer essa proteção.

---

## 18. Importante: não chamar de 2FA

O próprio projeto deixa claro que:

> **não é TOTP e não deve ser considerado um sistema de autenticação de dois fatores.**

fileciteturn1file1L242-L268

Portanto, a documentação deve usar termos como:

- **segunda camada de segredo**;
- **segredo complementar**;
- **senha complementar determinística**;
- **deterministic password companion**;
- **camada adicional sobre o gerenciador de senhas**.

E evitar afirmar que o projeto fornece:

- MFA padronizado;
- 2FA formal;
- TOTP;
- autenticação baseada em posse de dispositivo;
- segurança equivalente a uma chave FIDO2/WebAuthn.

---

## 19. Privacidade

A implementação foi projetada para funcionar localmente.

Segundo o código:

```text
Nome
Senha
PIN
Código
```

não são enviados para um servidor.

Também não há:

```text
Banco de dados
API externa
Backend
Serviço de terceiros
localStorage
sessionStorage
Math.random()
TOTP
```

A operação HMAC é realizada pela Web Crypto API do navegador. fileciteturn1file3L661-L703

Assim, a arquitetura atual é:

```text
Usuário
   ↓
Navegador
   ↓
JavaScript
   ↓
Web Crypto API
   ↓
Código
```

---

## 20. Atualização automática

Quando qualquer uma das entradas muda:

```text
Nome
Senha
PIN
```

uma nova geração é iniciada automaticamente. fileciteturn1file3L580-L612

---

## 21. Proteção contra cálculos antigos

Como o HMAC é calculado de forma assíncrona, podem existir vários cálculos em andamento.

O sistema utiliza `generationId`.

Exemplo:

```text
Geração 1 começa
      ↓
Usuário muda os dados
      ↓
Geração 2 começa
      ↓
Geração 2 termina
      ↓
Geração 1 termina depois
      ↓
Resultado da Geração 1 é descartado
```

Isso evita que um cálculo antigo sobrescreva o resultado mais recente. fileciteturn1file3L617-L656

---

## 22. Recursos da interface

A aplicação possui:

- modo escuro;
- modo claro;
- mostrar/ocultar senha;
- mostrar/ocultar PIN;
- validação do PIN;
- exibição do código;
- cópia do código;
- botão para limpar os campos;
- página explicativa;
- layout responsivo;
- suporte à redução de animações.

fileciteturn1file8L1537-L1647

---

## 23. Exemplo completo

Entradas:

```text
Nome:
Gerador

Senha:
senha123

PIN:
0123
```

Fluxo:

```text
Gerador
   ↓
gerador
   ↓
7 letras
3 vogais
4 consoantes
   ↓
PIN 0123
   ↓
01 | 2 | 3
   ↓
BASE = 48
   ↓
name=gerador|pin=0123|base=48
   ↓
senha123 como chave
   ↓
HMAC-SHA-256
   ↓
BigInt
   ↓
módulo 62¹²
   ↓
Base62
   ↓
12 caracteres
```

fileciteturn1file9L1658-L1708

---

## 24. Limitações de segurança

O projeto não deve substituir mecanismos profissionais de autenticação.

Principais limitações:

1. O código é determinístico.
2. Não há rotação automática baseada em tempo.
3. Não é TOTP.
4. Não é uma chave física de segurança.
5. Não é passkey/WebAuthn.
6. O PIN possui somente quatro dígitos.
7. A segurança depende da qualidade e do sigilo das entradas.
8. Um computador comprometido pode capturar os dados.
9. Um keylogger pode capturar o que o usuário digita.
10. O serviço utilizado precisa aceitar um segredo adicional para que esse modelo seja aplicável.

O próprio código alerta que o projeto não substitui mecanismos profissionais de autenticação, gerenciamento de credenciais ou sistemas de segurança específicos. fileciteturn1file1L242-L268

---

## 25. Declaração de desenvolvimento com IA

Este projeto foi desenvolvido **com auxílio de Inteligência Artificial**.

Informações de autoria:

```text
Autor do projeto:
maicofrone

Assistente de IA:
ChatGPT

Uso da IA:
Auxílio no desenvolvimento, estruturação,
implementação, explicação e documentação.
```

O próprio aplicativo possui uma identificação visual informando:

```text
Feito por maicofrone, usando ChatGPT
```

fileciteturn0file0L1187-L1194

Isso significa que a documentação e partes do desenvolvimento tiveram assistência de IA. O código não deve ser considerado automaticamente seguro apenas por utilizar algoritmos criptográficos conhecidos. Para utilização em contas realmente importantes, recomenda-se revisão humana e, idealmente, auditoria independente de segurança.

---

# Final Statement / Declaração final

This project should be understood as a **deterministic, local, AI-assisted password-companion tool intended to provide an additional secret layer alongside password managers such as 1Password, Bitwarden, and Proton Pass**.

The project, its explanations, and **this `.md` documentation file were created with assistance from AI (ChatGPT)**.

Este projeto deve ser entendido como uma **ferramenta determinística, local e desenvolvida com auxílio de IA, destinada a fornecer uma camada adicional de segredo em conjunto com gerenciadores de senhas como 1Password, Bitwarden e Proton Pass**.

O projeto, suas explicações e **este próprio arquivo `.md` foram criados com auxílio de IA (ChatGPT)**.

It is **not TOTP, not a formal 2FA implementation, and not a replacement for professional MFA mechanisms**.

Ele **não é TOTP, não é uma implementação formal de 2FA e não substitui mecanismos profissionais de MFA**.
