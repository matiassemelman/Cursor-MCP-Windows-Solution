# Solución para servidores MCP en Windows

## Contexto

El Model Context Protocol (MCP) es una forma de ofrecer nuevas herramientas al Cursor Agent. Los servidores MCP como Firecrawl-MCP (para scraping web) o GitHub-MCP (para interactuar con GitHub) presentan problemas específicos de ejecución en sistemas Windows.

## Pasos previos

Antes de configurar los servidores MCP, asegúrate de completar los siguientes pasos:

1. Activar el early access en Cursor Settings -> Beta -> Update Frequency. Esto nos da acceso a la última actualización que introduce lo siguiente: MCP: Added global server configuration with ~/.cursor/mcp.json and support for environment variables.

2. Actualizar a la versión 0.47.5

## Problema

Los servidores MCP no se mantienen activos en Windows, mostrando errores como "Client closed" poco después de iniciarse o "Authentication Failed: Bad credentials" al intentar usar las herramientas.

### Situación inicial

La configuración inicial intenta ejecutar el servidor directamente:

```json
{
  "mcpServers": {
    "firecrawl-mcp": {
      "command": "cmd",
      "args": [
        "/k",
        "set",
        "FIRECRAWL_API_KEY=tu_api_key",
        "&&",
        "npx",
        "-y",
        "firecrawl-mcp"
      ]
    }
  }
}
```

Esta configuración presenta problemas como "Client closed" o credenciales no reconocidas.

## Análisis del problema

### Diferencias entre Windows y sistemas Unix

En sistemas Windows:
1. Cuando un proceso padre (cmd.exe) termina, generalmente cierra todos sus procesos hijos
2. El flag `/c` en cmd.exe ejecuta el comando y luego termina la ventana inmediatamente
3. Node.js en Windows depende más del proceso que lo invoca para mantenerse vivo
4. Las variables de entorno configuradas con `set` dentro de un proceso cmd no están disponibles correctamente para el servidor MCP

En sistemas Unix:
1. Los procesos hijos pueden seguir ejecutándose incluso después de que el padre termina (comportamiento de "fork")
2. La gestión de procesos en segundo plano y variables de entorno es más nativa

### Causas raíz identificadas

1. En Windows, los comandos `npx` y `npm` son realmente scripts batch (`npx.cmd` y `npm.cmd`)
2. Los scripts batch requieren un intérprete (cmd.exe) para ejecutarse
3. El proceso no se mantiene vivo de forma independiente
4. Las credenciales (API keys, tokens) no se pasan correctamente al proceso del servidor MCP

## Solución implementada

La solución completa tiene dos partes:

1. Separar el proceso del servidor MCP usando el comando `start` de Windows
2. Proporcionar las credenciales a través del objeto `env` en la configuración JSON

### Para Firecrawl-MCP:

```json
{
  "mcpServers": {
    "firecrawl-mcp": {
      "command": "cmd.exe",
      "args": [
        "/c",
        "start",
        "/min",
        "cmd",
        "/k",
        "&& npx -y firecrawl-mcp"
      ],
      "env": {
        "FIRECRAWL_API_KEY": "tu_api_key"
      }
    }
  }
}
```

### Para GitHub-MCP:

```json
{
  "mcpServers": {
    "github": {
      "command": "cmd.exe",
      "args": [
        "/c",
        "start",
        "/min",
        "cmd",
        "/k",
        "&& npx -y @modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "tu_token_de_github"
      }
    }
  }
}
```