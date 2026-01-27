# About Model Context Protocol (MCP)

Model Context Protocol (MCP) is a protocol that allows you to extend the capabilities of GitHub Copilot by integrating it with other systems.

## Overview of Model Context Protocol (MCP)

The Model Context Protocol (MCP) is an open standard that defines how applications share context with large language models (LLMs). MCP provides a standardized way to connect AI models to different data sources and tools, enabling them to work together more effectively.

You can use MCP to extend the capabilities of Copilot Chat by integrating it with a wide range of existing tools and services. For example, the GitHub MCP server allows you to use Copilot Chat in your IDE to perform tasks on GitHub. You can also use MCP to create new tools and services that work with Copilot Chat, allowing you to customize and enhance your experience.

For more information on MCP, see [the official MCP documentation](https://modelcontextprotocol.io/introduction). For information on currently available MCP servers, see [the MCP servers repository](https://github.com/modelcontextprotocol/servers/tree/main).

To learn how to configure and use MCP servers with Copilot Chat, see [Extending GitHub Copilot Chat with Model Context Protocol (MCP) servers](/en/copilot/how-tos/context/model-context-protocol/extending-copilot-chat-with-mcp).

Enterprises and organizations can choose to enable or disable use of MCP for members of their organization or enterprise with the **MCP servers in Copilot** policy. The policy is disabled by default. See [Managing policies and features for GitHub Copilot in your enterprise](/en/copilot/how-tos/administer/enterprises/managing-policies-and-features-for-copilot-in-your-enterprise) and [Managing policies and features for GitHub Copilot in your organization](/en/copilot/how-tos/administer-copilot/manage-for-organization/manage-policies). The MCP policy **only** applies to users who have a Copilot Business or Copilot Enterprise subscription from an organization or enterprise that configures the policy. Copilot Free, Copilot Pro, or Copilot Pro+ **do not** have their MCP access governed by this policy.

## Availability

There is currently broad support for local MCP servers in clients such as Visual Studio Code, JetBrains IDEs, XCode, and others.

Support for remote MCP servers is growing, with editors like Visual Studio Code, Visual Studio, JetBrains IDEs, Xcode, Eclipse, and Cursor providing this functionality with OAuth or PAT, and Windsurf supporting PAT only.

To find out if your preferred editor supports remote MCP servers, check the documentation for your specific editor.

## About the GitHub MCP server

The GitHub MCP server is a Model Context Protocol (MCP) server provided and maintained by GitHub.

GitHub MCP server can be used to:

* Automate and streamline code-related tasks.
* Connect third-party tools (like Cursor, Windsurf, or future integrations) to leverage GitHub’s context and AI capabilities.
* Enable cloud-based workflows that work from any device, without local setup.
* Invoke GitHub tools, such as Copilot coding agent (requires GitHub Copilot subscription) and code scanning (requires GitHub Advanced Security subscription), to assist with code generation and security analysis.

To learn how to set up and use the GitHub MCP server, see [Using the GitHub MCP Server](/en/copilot/how-tos/context/model-context-protocol/using-the-github-mcp-server).

### Remote access

You can access the GitHub MCP server remotely through Copilot Chat in Visual Studio Code without any local setup. The remote server has access to additional toolsets only available in the remote GitHub MCP server. For a list of such tools, see [Additional toolsets](https://github.com/github/github-mcp-server?tab=readme-ov-file#additional-toolsets-in-remote-github-mcp-server) in the `github/github-mcp-server` repository.

The GitHub MCP server can also run locally in any MCP-compatible editor, if necessary.

### Toolset customization

> \[!IMPORTANT]
> Always review the GitHub MCP server repository at [github/github-mcp-server](https://github.com/github/github-mcp-server) for the latest toolsets and authoritative configuration guidance.

The GitHub MCP server supports enabling or disabling specific groups of functionalities via toolsets. Toolsets allow you to control which GitHub API capabilities are available to your AI tools.

Enabling only the toolsets you need improves your AI assistant's performance and security. Fewer tools means better tool selection accuracy and fewer errors. Disabling unused toolsets also frees up tokens in the AI's context window.

Toolsets do not only include tools, but also relevant MCP resources and prompts where applicable.

To learn how to configure toolsets for the GitHub MCP server, see [Configuring toolsets for the GitHub MCP Server](/en/copilot/how-tos/context/use-mcp/configure-toolsets).

### Security

For all public repositories, and private repositories covered by GitHub Advanced Security, interactions with the GitHub MCP server are secured by push protection, which blocks secrets from being included in AI-generated responses and prevents you from exposing secrets through any actions you perform using the server, such as creating an issue. For more information, see [Working with push protection and the GitHub MCP server](/en/code-security/secret-scanning/working-with-secret-scanning-and-push-protection/working-with-push-protection-and-the-github-mcp-server).

## About the GitHub MCP Registry

The GitHub MCP Registry is a curated list of MCP servers from partners and the community. You can use the registry to discover new MCP servers and find ones that meet your specific needs. See [the GitHub MCP Registry](https://github.com/mcp).

> \[!NOTE]
> The GitHub MCP Registry is currently in public preview and subject to change.

## Next steps

* [Extending GitHub Copilot Chat with Model Context Protocol (MCP) servers](/en/copilot/how-tos/context/model-context-protocol/extending-copilot-chat-with-mcp)
* [Using the GitHub MCP Server](/en/copilot/how-tos/context/model-context-protocol/using-the-github-mcp-server)
* [Enhancing GitHub Copilot agent mode with MCP](/en/copilot/tutorials/enhancing-copilot-agent-mode-with-mcp)