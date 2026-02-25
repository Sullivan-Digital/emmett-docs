---
tags:
  - utility
  - config
aliases:
  - Emmett Plugins
  - emmett.config.ts
related:
  - "[[Utilities MOC]]"
package: emmett
---

# Plugin System

Emmett has a declarative plugin system currently focused on CLI extensions. Plugins are configured declaratively and register commands with the Emmett CLI.

> [!note] The plugin system currently only supports the `'cli'` plugin type. There is no runtime plugin discovery, hook system, or middleware chain. Plugins are intended for extending the Emmett CLI with custom commands.

## Plugin Configuration

```typescript
import { type EmmettPluginsConfig, type EmmettPluginConfig } from '@event-driven-io/emmett';

const config: EmmettPluginsConfig = {
  plugins: [
    // Simple form: just a plugin name
    'my-plugin',

    // Full form: name with registrations
    {
      name: 'my-plugin',
      register: [
        { pluginType: 'cli', path: './plugins/my-cli-plugin' },
      ],
    },
  ],
};
```

## Plugin Implementation

```typescript
import { type EmmettCliPlugin, type EmmettCliCommand } from '@event-driven-io/emmett';

const myPlugin: EmmettCliPlugin = {
  pluginType: 'cli',
  name: 'my-custom-commands',
  registerCommands: (program: EmmettCliCommand) => {
    program.addCommand(/* your CLI command definition */);
  },
};
```

## Type Checking

```typescript
import { isPluginConfig } from '@event-driven-io/emmett';

if (isPluginConfig(maybePlugin)) {
  // maybePlugin is a valid EmmettPluginConfig
}
```

## Types

```typescript
type EmmettPluginsConfig = { plugins: EmmettPluginConfig[] };

type EmmettPluginConfig =
  | { name: string; register: EmmettPluginRegistration[] }
  | string;

type EmmettPluginType = 'cli';
type EmmettCliPluginRegistration = { pluginType: 'cli'; path?: string };
type EmmettPluginRegistration = EmmettCliPluginRegistration;

type EmmettCliPlugin = {
  pluginType: 'cli';
  name: string;
  registerCommands: (program: EmmettCliCommand) => Promise<void> | void;
};

type EmmettPlugin = EmmettCliPlugin;
```

## See Also

- [[Utilities MOC]] -- Other utility modules
