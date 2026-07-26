# Push notifications — FidelizaAi / CarStatus (wrappers Capacitor)

Branch `push-notifications` nos dois repos (`cap-fidelizaai-ios`, `cap-carstatus-ios`).
NÃO mergear na `main` enquanto a review 1.0 estiver rolando: push na main dispara build
no Codemagic e um binário novo cancelaria a submissão em análise.

## Por que precisa de app nativo
O iOS não entrega Web Push dentro de WKWebView (só Safari/PWA na tela de início). Como os
dois apps são WebView do site, notificação só sai pelo caminho nativo: plugin Capacitor →
APNs → aparelho.

## O que já está feito nesta branch
- `@capacitor/push-notifications@8.1.2` instalado e sincronizado no projeto iOS.
- `ios/App/App/App.entitlements` com `aps-environment = production`.
- `CODE_SIGN_ENTITLEMENTS` apontado nas duas configurações (Debug/Release) do Xcode.

## O que falta — depende de ação no portal Apple (não tem API pública)
1. **Chave APNs (.p8)**: Certificates, Identifiers & Profiles → Keys → `+` → marcar
   *Apple Push Notifications service (APNs)* → baixar o `.p8` (download único) e anotar o
   **Key ID** e o **Team ID** (9R57QQ98N6). Guardar em `/opt/NIAR/credentials/apns/`.
   Uma única chave serve para todos os apps da conta.
2. **Capability Push no App ID** dos dois bundles. Isso *dá* para fazer pela API
   (`POST /v1/bundleIdCapabilities`, capabilityType `PUSH_NOTIFICATIONS`), mas só fazer
   quando for gerar o build novo — mexer no App ID durante a review não é necessário.

## O que falta — do nosso lado (código)
1. **Site**: chamar o registro quando rodando dentro do app (snippet abaixo) e mandar o
   token pro backend.
2. **Backend**: tabela de tokens (`device_tokens`: token, plataforma, user_id, criado_em,
   último_visto) + endpoint de registro.
3. **Disparador**: serviço que assina um JWT ES256 com a chave APNs e faz POST em
   `https://api.push.apple.com/3/device/<token>` (HTTP/2). Sem custo, roda na VPS.

### Snippet do site (funciona em qualquer um dos dois)
O Capacitor injeta a ponte nativa mesmo carregando o site remoto, então o próprio site
pode chamar o plugin:

```js
export async function registrarPush(salvarToken) {
  const Cap = window.Capacitor;
  if (!Cap?.isNativePlatform?.()) return;            // no navegador, não faz nada
  const Push = Cap.Plugins.PushNotifications;

  const perm = await Push.requestPermissions();
  if (perm.receive !== 'granted') return;            // usuário recusou

  Push.addListener('registration', ({ value }) => salvarToken(value, 'ios'));
  Push.addListener('registrationError', (e) => console.warn('push registration', e));
  await Push.register();
}
```

Chamar depois do login (o token precisa ficar amarrado ao usuário para o disparo ser
direcionado).
