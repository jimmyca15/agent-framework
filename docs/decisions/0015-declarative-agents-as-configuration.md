---
status: proposed
contact: tbd
date: 2026-01-29
deciders: tbd
consulted:
informed:
---

# Declarative Agents as Configuration (YAML in configuration systems)

## Context and Problem Statement

The Agent Framework supports defining agents using a declarative YAML spec. Today, developers typically keep that YAML in a file checked into source control and load it at startup.

Many application stacks (notably .NET and Python) already have mature configuration systems (environment-specific settings, centralized configuration, live reload). We want to enable developers to store agent YAML in those configuration systems and create agents from it in a first-class, ergonomic way.

This ADR is specifically about **how YAML agent specs integrate with configuration**, and whether the framework should support **runtime updates** (hot swapping) as a first-class feature.

## Decision Drivers

- Developer experience: simple "getting started" path that looks like other configuration-bound components.
- Operational agility: ability to update instructions and other non-secret parameters quickly (e.g., incident response / livesite).
- Environment support: easy separation of dev/staging/prod behavior using existing configuration layering.
- Safety: avoid accidentally encouraging secrets in YAML; preserve secure secret management patterns.
- Consistency: keep a coherent story across .NET and Python without requiring provider-specific knowledge.
- Reliability: if runtime updates are supported, define clear semantics (thread safety, in-flight requests, resource disposal).

## Examples

### Storing YAML in configuration (.NET `appsettings.json`)

Conceptually:

```json
{
  "AgentSpec": "{Agent yaml}"
}
```

Note: JSON string escaping and multiline handling are implementation details of the host configuration system. The intent is that the YAML is retrievable as a string (or equivalent value) from configuration.

### Live update scenario (concept)

Configuration enables live update scenarios. In .NET there is a change token to subscribe to; in Python, configuration libraries often support callbacks or reload hooks.

Potential benefits of live updates:

- Avoid startup time
- Avoid shutdown time
- Avoid memory cache breaks

### Before

Assume spec:

```yaml
kind: Prompt
name: Greeter
description: You are an assistant who greets people.
instructions: You always reply with "Hello!"
```

Assume code:

```csharp
// Assume you already have an IChatClient.
PromptAgentFactory agentFactory = new ChatClientPromptAgentFactory(chatClient);

AIAgent agent = await agentFactory.CreateFromYamlAsync(configuration["AgentSpec"]);

Console.WriteLine(await agent.RunAsync("Hi"));
```

Expected output: `Hello!`

### After

Assume configuration updated. The instructions have changed, the spec is now:

```yaml
kind: Prompt
name: Greeter
description: You are an assistant who greets people.
instructions: You always reply with "Greetings!" # Hello updated to Greetings
```

Code is unchanged.

Expected output: `Greetings!`

## Considered Options

- Option 1: Use existing YAML entry points (no new APIs), no runtime updates
- Option 2: Add helper API to bind YAML from configuration, no runtime updates
- Option 3: Add configuration-bound provider that supports runtime updates (hot swapping)

## Pros and Cons of the Options

### Option 1: Use existing YAML support (no new APIs), no runtime updates

Use the configuration system to retrieve the YAML as a string and call existing APIs.

Example:

```csharp
PromptAgentFactory agentFactory = new ChatClientPromptAgentFactory(chatClient);

AIAgent agent = await agentFactory.CreateFromYamlAsync(configuration["AgentSpec"]);
```

- Good, because it requires no new framework APIs.
- Good, because it keeps the framework surface area small.
- Neutral, because it already works for many applications.
- Bad, because it doesn't clearly communicate the intended "configuration-first" pattern.
- Bad, because it offers no guidance on best practices (naming, layering, validation).

### Option 2: Add helper API to bind YAML from configuration, no runtime updates

Add a helper that makes the configuration scenario explicit.

Proposed API shape (.NET):

```csharp
public static class PromptAgentFactoryConfigurationExtensions
{
  public static Task<AIAgent> CreateFromConfigurationAsync(this PromptAgentFactory agentFactory, IConfigurationSection section, CancellationToken cancellationToken = default);
}
```

Usage:

```csharp
PromptAgentFactory agentFactory = new ChatClientPromptAgentFactory(chatClient);

AIAgent agent = await agentFactory.CreateFromConfigurationAsync(configuration.GetSection("AgentSpec"));
```

- Good, because the intended configuration scenario is obvious and discoverable.
- Good, because it allows consistent validation/errors (e.g., missing section, empty YAML).
- Neutral, because it still relies on the same underlying YAML parsing path.
- Bad, because it adds API surface area that must be maintained.

### Option 3: Configuration-bound provider with runtime updates (hot swapping)

Add a configuration-bound provider that updates the underlying agent when configuration changes.

Proposed API shape (.NET):

```csharp
public interface IAIAgentProvider
{
  ValueTask<AIAgent> GetAgentAsync(CancellationToken cancellationToken = default);
}

public static class PromptAgentFactoryConfigurationExtensions
{
  public static IAIAgentProvider CreateProviderFromConfiguration(this PromptAgentFactory agentFactory, IConfigurationSection section);
}
```

Usage:

```csharp
PromptAgentFactory agentFactory = new ChatClientPromptAgentFactory(chatClient);

IAIAgentProvider agentProvider = agentFactory.CreateProviderFromConfiguration(configuration.GetSection("AgentSpec"));

// Prints "Hello!"
Console.WriteLine(await (await agentProvider.GetAgentAsync()).RunAsync("Hi"));

// Update configuration before continuing
_ = Console.ReadLine();

// Prints "Greetings!"
Console.WriteLine(await (await agentProvider.GetAgentAsync()).RunAsync("Hi"));
```

Under the hood, the agent would be hot swapped on configuration reload.

Naturally, this should extend to modification of other agent properties such as endpoint and model id, via usage of `AggregatorAgentFactory`

```csharp
var agentFactory = new AggregatorAgentFactory(
    [
        new OpenAIChatAgentFactory(endpointUri, tokenCredential),
        new OpenAIResponseAgentFactory(endpointUri, tokenCredential),
        new OpenAIAssistantAgentFactory(endpointUri, tokenCredential)
    ]);

//
// Set up so that the agent can handle runtime updates to model id
IAIAgentProvider agentProvider = agentFactory.CreateProviderFromConfiguration(configuration.GetSection("AgentSpec"));
```

### IAIAgentProvider or AIAgent directly

The first pass at the interface yielded the IAIAgentProvider concept which could provider a new AIAgent given a runtime configuration update. However, another option is to re-use the same AIAgent instance and update AIAgent to be able to update in place

```csharp
public static class PromptAgentFactoryConfigurationExtensions
{
  public static AIAgent CreateProviderFromConfiguration(this PromptAgentFactory agentFactory, IConfigurationSection section);
}
```

- Good, because it enables true live-update scenarios without app-level glue code.
- Good, because it reduces operational friction for instruction tweaks.
- Neutral, because it requires careful definition of lifecycle semantics.
- Bad, because it introduces concurrency/lifecycle complexity (thread safety, in-flight requests, disposing old instances).
- Bad, because it requires more extensive tests and potentially new abstractions.

## Decision Outcome

TBD