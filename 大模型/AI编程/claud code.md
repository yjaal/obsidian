
安装
```
brew install --cask claude-code
```

模型配置 `~/.zshrc`

```
# 硅基流动模型广场
export ANTHROPIC_BASE_URL=https://api.siliconflow.cn/
export ANTHROPIC_AUTH_TOKEN=sk-kdvhqihyfiaolpm
```

然后在 `~/.claude/settings.json` 配置具体的模型

```json
{
  "env": {
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "Pro/zai-org/GLM-5",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "Pro/MiniMaxAI/MiniMax-M2.5",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "Pro/moonshotai/Kimi-K2.5",
    "ANTHROPIC_MODEL": "Pro/moonshotai/Kimi-K2.5"
  }
}
```










