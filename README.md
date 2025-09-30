# Guia Completo: Notificações Push com Deep Linking (Expo + Spring Boot)

Este guia detalha o passo a passo para implementar notificações push que, ao serem clicadas, abrem uma tela específica dentro do seu aplicativo (deep linking). Usaremos um frontend em **React Native com Expo** e um backend em **Java com Spring Boot**.

## Arquitetura Geral

O fluxo de dados funciona da seguinte forma:

1.  **App Expo** solicita permissão e obtém um `ExpoPushToken` único do dispositivo.
2.  **App Expo** envia esse token para o nosso **Backend Java Spring**.
3.  **Backend** armazena o token, associando-o a um usuário.
4.  Quando um evento ocorre, o **Backend** monta uma mensagem e a envia para a **API do Expo**, junto com o token do destinatário e um link (ex: `meuapp://produtos/123`).
5.  A **API do Expo** entrega a notificação ao dispositivo correto (iOS ou Android).
6.  O usuário clica na notificação, e o **App Expo** usa o link para navegar até a tela do produto 123.

## Parte 1: Configuração no Frontend (Expo)

### Passo 1.1: Instalar Dependências

No terminal, na raiz do seu projeto Expo, execute:

```bash
npx expo install expo-notifications expo-device expo-linking
```

* `expo-notifications`: Para lidar com tudo relacionado a notificações.
* `expo-device`: Para verificar se o app está rodando em um dispositivo físico.
* `expo-linking`: Para gerenciar os deep links.

### Passo 1.2: Configurar o Deep Link (URI Scheme)

Para que o sistema operacional saiba qual app abrir a partir de um link como `meuapp://`, precisamos registrar um "scheme". No seu arquivo `app.json`, adicione a chave `scheme`:

```json
{
  "expo": {
    "name": "Meu App",
    "slug": "meu-app",
    "scheme": "meuapp"
  }
}
```

> **Importante:** Se o app já estiver instalado, você precisará desinstalá-lo e reinstalá-lo para que o novo scheme seja registrado no sistema operacional.

### Passo 1.3: Obter o Token e Enviar ao Backend

Crie uma função para gerenciar o registro de notificações. O ideal é chamá-la após o login do usuário.

```javascript
// Arquivo: services/notificationService.js (exemplo)

import * as Device from 'expo-device';
import * as Notifications from 'expo-notifications';
import { Platform } from 'react-native';

// Configura como a notificação se comporta com o app aberto
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: false,
  }),
});

// Função principal para registrar o dispositivo
export async function registerForPushNotificationsAsync() {
  if (!Device.isDevice) {
    console.warn('Push notifications só funcionam em dispositivos físicos.');
    return null;
  }

  const { status } = await Notifications.requestPermissionsAsync();
  if (status !== 'granted') {
    alert('Permissão para notificações negada!');
    return null;
  }

  try {
    const token = (await Notifications.getExpoPushTokenAsync({
      // Encontre seu projectId no app.json em "extra.eas.projectId"
      projectId: 'SEU_PROJECT_ID_AQUI',
    })).data;

    console.log('Expo Push Token:', token);

    // **AÇÃO:** Envie o token para o seu backend aqui.
    // await sendTokenToBackend(token);

    if (Platform.OS === 'android') {
      await Notifications.setNotificationChannelAsync('default', {
        name: 'default',
        importance: Notifications.AndroidImportance.MAX,
      });
    }

    return token;
  } catch (error) {
    console.error('Erro ao obter o push token:', error);
    return null;
  }
}
```

### Passo 1.4: Lidar com o Clique na Notificação (Deep Link)

No componente principal da sua aplicação (como o `App.js`), adicione um listener para capturar a interação do usuário com a notificação.

```javascript
// Arquivo: App.js

import { useEffect } from 'react';
import * as Notifications from 'expo-notifications';
import * as Linking from 'expo-linking';
import { NavigationContainer } from '@react-navigation/native'; // Exemplo com React Navigation

export default function App() {
  useEffect(() => {
    // Listener ativado quando um usuário clica em uma notificação
    const subscription = Notifications.addNotificationResponseReceivedListener(response => {
      const data = response.notification.request.content.data;

      if (data && data.url) {
        console.log('Notificação clicada! Abrindo deep link:', data.url);
        // Usa o expo-linking para navegar para a URL recebida
        Linking.openURL(data.url);
      }
    });

    return () => subscription.remove();
  }, []);

  // Para que Linking.openURL funcione, seu sistema de navegação (ex: React Navigation)
  // precisa estar configurado para lidar com deep links.
  const linkingConfig = {
    prefixes: [Linking.createURL('/')], // Usa o scheme do app.json (ex: meuapp://)
    config: {
      screens: {
        Produto: 'produtos/:id', // Mapeia a rota 'produtos/:id' para a tela 'Produto'
        // ... outras telas
      }
    }
  };

  return <NavigationContainer linking={linkingConfig} />;
}
```

## Parte 2: Configuração no Backend (Java Spring)

### Passo 2.1: Endpoint para Salvar o Token

Crie um endpoint para receber o `ExpoPushToken` do app e salvá-lo no banco de dados.

```java
// DTO para receber o token
@Data // Lombok
public class PushTokenDto {
    private String pushToken;
}

@RestController
@RequestMapping("/api/dispositivos")
public class DispositivoController {

    @PostMapping("/registrar")
    public ResponseEntity<Void> registrarDispositivo(@RequestBody PushTokenDto dto) {
        // **AÇÃO:** Implemente a lógica para salvar o dto.getPushToken()
        // associando-o ao usuário autenticado.
        System.out.println("Registrando token: " + dto.getPushToken());
        return ResponseEntity.ok().build();
    }
}
```

### Passo 2.2: Lógica para Enviar a Notificação com Deep Link

Crie um serviço para montar e enviar a requisição para a API do Expo. A chave para o deep link é o campo `data`.

```java
// DTO para a mensagem de notificação
@Data
@AllArgsConstructor
public class PushMessage {
    private String to;      // O "ExponentPushToken[...]"
    private String title;
    private String body;
    private Map<String, Object> data; // Campo para dados extras (nosso deep link)
}

@Service
public class ExpoNotificationService {

    private final RestTemplate restTemplate = new RestTemplate();
    private static final String EXPO_PUSH_URL = "[https://exp.host/--/api/v2/push/send](https://exp.host/--/api/v2/push/send)";

    public void sendNotificationWithDeepLink(String recipientToken, String title, String message, String url) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);

        // Cria o mapa de dados com a URL do deep link
        Map<String, Object> dataMap = Collections.singletonMap("url", url); // Ex: "meuapp://produtos/123"

        PushMessage notification = new PushMessage(recipientToken, title, message, dataMap);

        List<PushMessage> notifications = Collections.singletonList(notification);
        HttpEntity<List<PushMessage>> request = new HttpEntity<>(notifications, headers);

        try {
            String response = restTemplate.postForObject(EXPO_PUSH_URL, request, String.class);
            System.out.println("Resposta da API Expo: " + response);
        } catch (HttpClientErrorException e) {
            System.err.println("Erro ao enviar notificação: " + e.getResponseBodyAsString());
        }
    }
}
```

## Parte 3: Configurando o Firebase (FCM) para Android - Obrigatório para Builds

> **Quando isso é necessário?**
>
> * **NÃO precisa** se você testa apenas no app **Expo Go** (`expo dev`).
> * **PRECISA** se você vai gerar um app próprio (`.apk`) com `eas build`.

### Passo 3.1: Criar e Configurar um Projeto no Firebase

1.  Acesse o [Console do Firebase](https://console.firebase.google.com/) e crie um novo projeto.
2.  Dentro do projeto, clique no ícone do **Android** (</>) para adicionar um app.
3.  No campo **"Nome do pacote Android"**, insira o valor exato da chave `android.package` do seu `app.json` (ex: `com.seuusuario.seuapp`).
4.  Clique em **"Registrar app"**.

### Passo 3.2: Adicionar o Arquivo de Configuração ao Projeto

1.  Na próxima etapa, clique em **"Fazer o download de google-services.json"**.
2.  Copie este arquivo `google-services.json` para a **raiz do seu projeto Expo**.
3.  No seu `app.json`, adicione a seguinte linha dentro do objeto `android`:

    ```json
    "googleServicesFile": "./google-services.json"
    ```

### Passo 3.3: Fazer Upload das Credenciais do Servidor para o Expo (EAS)

1.  No Console do Firebase, vá em **Configurações do Projeto** > **Contas de serviço**.
2.  Clique em **"Gerar nova chave privada"**. Um arquivo JSON será baixado.
3.  No terminal, na raiz do seu projeto, rode `eas credentials`.
4.  Siga os passos: escolha `android`, e quando for solicitado pela **"FCM V1 service account key"**, abra o arquivo JSON que você baixou, copie todo o seu conteúdo e cole no terminal.

**Pronto!** Seu projeto está configurado para enviar notificações para builds próprios do seu aplicativo. Para iOS, o processo é similar e o `eas build` irá guiá-lo na configuração das credenciais da Apple (APNs).

---

## Referências

* **Documentação Oficial do Expo Notifications:** [expo-notifications](https://docs.expo.dev/versions/latest/sdk/notifications/)
* **Documentação Oficial do Expo Linking:** [expo-linking](https://docs.expo.dev/versions/latest/sdk/linking/)
* **Console do Firebase:** [Firebase Console](https://console.firebase.google.com/)
* **Exemplo de Modelagem do Banco de Dados:** [dbdiagram.io](https://dbdiagram.io/d/68dbbe47d2b621e42294c532)
* **Ferramenta Expo para teste de notificações:** [Push notifications tool](https://expo.dev/notifications)
