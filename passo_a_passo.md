Aqui estão as classes que precisam ser alteradas, com seus caminhos completos e as correções necessárias para mitigar a CVE-2024-38819:

1. Classes a serem alteradas e caminhos completos

a) RouterFunction
Spring Web MVC:
spring-webmvc/src/main/java/org/springframework/web/servlet/function/RouterFunction.java
Spring WebFlux:
spring-webflux/src/main/java/org/springframework/web/reactive/function/server/RouterFunction.java
b) HandlerFunction
Spring Web MVC:
spring-webmvc/src/main/java/org/springframework/web/servlet/function/HandlerFunction.java
Spring WebFlux:
spring-webflux/src/main/java/org/springframework/web/reactive/function/server/HandlerFunction.java
c) ResourceHttpRequestHandler
Spring Web MVC:
spring-webmvc/src/main/java/org/springframework/web/servlet/resource/ResourceHttpRequestHandler.java
2. Correções a serem feitas

a) Adicionar Validação de Caminhos
Implemente uma função que garanta que os caminhos solicitados estejam restritos a diretórios permitidos. Essa função será usada em todos os métodos que manipulam caminhos.

Exemplo:

import java.nio.file.Path;
import java.nio.file.Paths;

public static boolean isPathSecure(String basePath, String requestedPath) {
    try {
        Path base = Paths.get(basePath).toRealPath();
        Path resolved = Paths.get(basePath, requestedPath).toRealPath();
        return resolved.startsWith(base); // Garante que o caminho está dentro do diretório base
    } catch (Exception e) {
        return false; // Caminho inválido ou fora do permitido
    }
}
b) Sanitizar a Entrada
Crie uma função para sanitizar o caminho solicitado, removendo caracteres suspeitos como ../.

Exemplo:

public static String sanitizePath(String path) {
    return path.replaceAll("[^a-zA-Z0-9._/-]", "_"); // Remove caracteres inválidos
}
c) Alterar Métodos Específicos
1. resolveResource em ResourceHttpRequestHandler

Adicione validação de caminhos antes de acessar os arquivos.

@Override
protected Resource resolveResource(HttpServletRequest request, String requestPath, List<? extends Resource> locations, ResourceResolverChain chain) {
    String sanitizedPath = sanitizePath(requestPath); // Sanitiza o caminho
    if (!isPathSecure(BASE_PATH, sanitizedPath)) {
        throw new SecurityException("Tentativa de acesso não autorizado detectada!"); // Bloqueia acesso não autorizado
    }
    return super.resolveResource(request, sanitizedPath, locations, chain); // Continua a execução normal
}
2. apply em RouterFunction

Adicione validação de segurança ao processar a requisição.

@Override
public ServerResponse apply(ServerRequest request) {
    String path = sanitizePath(request.path());
    if (!isPathSecure(BASE_PATH, path)) {
        return ServerResponse.status(HttpStatus.FORBIDDEN).body("Acesso negado");
    }
    return this.handlerFunction.apply(request);
}
3. handle em HandlerFunction

Adicione validação antes de manipular a requisição.

@Override
public ServerResponse handle(ServerRequest request) {
    String sanitizedPath = sanitizePath(request.path());
    if (!isPathSecure(BASE_PATH, sanitizedPath)) {
        throw new SecurityException("Tentativa de acesso não autorizado detectada!");
    }
    return this.handlerFunction.apply(request);
}
3. Resumo

Classes e Métodos Alterados
RouterFunction (MVC e WebFlux):
Método: apply
Adicionar validação de caminho e sanitização de entrada.
HandlerFunction (MVC e WebFlux):
Método: handle
Adicionar validação de caminho e sanitização de entrada.
ResourceHttpRequestHandler (MVC):
Método: resolveResource
Adicionar validação de caminho e sanitização de entrada.
Com essas mudanças, sua aplicação estará protegida contra ataques de path traversal relacionados a esta vulnerabilidade. Se precisar de ajuda para implementar essas alterações, é só pedir!
