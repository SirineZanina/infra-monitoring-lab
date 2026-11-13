# Secrets (not committed)

Create `slack_webhook_url` in this folder, containing only your Slack incoming-webhook URL:

    echo 'https://hooks.slack.com/services/XXX/YYY/ZZZ' > secrets/slack_webhook_url
    chmod 600 secrets/slack_webhook_url

Alertmanager reads it through `slack_api_url_file`, so the URL never appears in the configuration.
