---
title: Notification template examples
sort_rank: 8
---

The following are all different examples of alerts and corresponding Alertmanager configuration file setups (alertmanager.yml).
Each use the [Go templating](http://golang.org/pkg/text/template/) system.

## Customizing Slack notifications

In this example we've customised our Slack notification to send a URL to our organisation's wiki on how to deal with the particular alert that's been sent.

```
global:
  # Also possible to place this URL in a file.
  # Ex: `slack_api_url_file: '/etc/alertmanager/slack_url'`
  slack_api_url: '<slack_webhook_url>'

route:
  receiver: 'slack-notifications'
  group_by: [alertname, datacenter, app]

receivers:
- name: 'slack-notifications'
  slack_configs:
  - channel: '#alerts'
    text: 'https://internal.myorg.net/wiki/alerts/{{ .GroupLabels.app }}/{{ .GroupLabels.alertname }}'
```

## Accessing annotations in CommonAnnotations

In this example we again customize the text sent to our Slack receiver accessing the `summary` and `description` stored in the `CommonAnnotations` of the data sent by the Alertmanager.

Alert

```
groups:
- name: Instances
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 5m
    labels:
      severity: page
    # Prometheus templates apply here in the annotation and label fields of the alert.
    annotations:
      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.'
      summary: 'Instance {{ $labels.instance }} down'
```

Receiver

```
- name: 'team-x'
  slack_configs:
  - channel: '#alerts'
    # Alertmanager templates apply here.
    text: "<!channel> \nsummary: {{ .CommonAnnotations.summary }}\ndescription: {{ .CommonAnnotations.description }}"
```

## Ranging over all received Alerts

Finally, assuming the same alert as the previous example, we customize our receiver to range over all of the alerts received from the Alertmanager, printing their respective annotation summaries and descriptions on new lines.

Receiver

```
- name: 'default-receiver'
  slack_configs:
  - channel: '#alerts'
    title: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}"
    text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
```

## Defining reusable templates

Going back to our first example, we can also provide a file containing named templates which are then loaded by Alertmanager in order to avoid complex templates that span many lines.
Create a file under `/alertmanager/template/myorg.tmpl` and create a template in it named "slack.myorg.text":

```
{{ define "slack.myorg.text" }}https://internal.myorg.net/wiki/alerts/{{ .GroupLabels.app }}/{{ .GroupLabels.alertname }}{{ end}}
```

The configuration now loads the template with the given name for the "text" field and we provide a path to our custom template file:

```
global:
  slack_api_url: '<slack_webhook_url>'

route:
  receiver: 'slack-notifications'
  group_by: [alertname, datacenter, app]

receivers:
- name: 'slack-notifications'
  slack_configs:
  - channel: '#alerts'
    text: '{{ template "slack.myorg.text" . }}'

templates:
- '/etc/alertmanager/templates/myorg.tmpl'
```

This example is explained in further detail in this [blogpost](https://prometheus.io/blog/2016/03/03/custom-alertmanager-templates/).

## Defining a Slack Block Kit template

A Slack receiver can send [Block Kit](https://docs.slack.dev/block-kit/) blocks instead of the default attachment by setting `blocks` to a template that renders a JSON array of blocks.
Unlike posting Block Kit with the webhook receiver, this works with the Slack-specific options such as `update_message` and `post_updates_to_thread`.

Create `/etc/alertmanager/templates/slack-blocks.tmpl`:

```gotemplate
{{ define "slack.myorg.blocks" }}
[
  {
    "type": "header",
    "text": {
      "type": "plain_text",
      "text": {{ printf "[%s] %s" (.Status | toUpper) .CommonLabels.alertname | toJson }}
    }
  }
  {{- range $i, $alert := .Alerts.Firing }}{{ if lt $i 10 }},
  {
    "type": "section",
    "text": {
      "type": "mrkdwn",
      "text": {{ or $alert.Annotations.summary $alert.Labels.alertname | toJson }}
    }
  }
  {{- end }}{{ end }}
]
{{ end }}
```

Reference the template from the receiver:

```yaml
receivers:
- name: 'slack-blocks'
  slack_configs:
  - channel: '#alerts'
    message_text: '{{ template "slack.default.fallback" . }}'
    blocks: '{{ template "slack.myorg.blocks" . }}'

templates:
- '/etc/alertmanager/templates/slack-blocks.tmpl'
```

Things to keep in mind:

* Every interpolated value must be piped through `toJson`. Otherwise a quote or newline in the alert data breaks the JSON and the notification fails.
* Set `message_text`. Slack uses the top-level text as the fallback for notifications and screen readers when a message has blocks.
* The template must render 1 to 50 blocks, otherwise the notification fails. Slack also limits the length of text in blocks, for example 3000 characters for a section and 150 for a header, so cap the number of alerts rendered in a range as in the example above.
* Fields of `.CommonAnnotations` and `.CommonLabels` are empty when the alerts in a group have different values, and Slack rejects empty text, so add a fallback with `or` as in the example above.
* All the attachment options of the Slack receiver, such as `title`, `text`, `color`, `fields` and `actions`, are ignored when `blocks` is set. The blocks are sent at the top level of the message, so there is no attachment color bar.
* With `update_message`, switching a receiver from `blocks` back to attachments while a message is still being updated leaves the old blocks above the new attachment, unless `message_text` is set. Switching to `blocks` replaces the old attachment.
* Alert data is not escaped for Slack `mrkdwn`, so text such as `<!channel>` in an annotation is rendered as a mention.

## Defining a reusable HTML email template

The HTML body of an email notification can use a named template from an external file.
Create `/etc/alertmanager/templates/email.tmpl` with the following content:

```html
{{ define "email.myorg.html" }}
<!DOCTYPE html>
<html>
  <body>
    <h1>{{ .Status | toUpper }}</h1>
    <table>
      <thead>
        <tr>
          <th>Alert</th>
          <th>Summary</th>
        </tr>
      </thead>
      <tbody>
        {{ range .Alerts }}
        <tr>
          <td>{{ index .Labels "alertname" }}</td>
          <td>{{ index .Annotations "summary" }}</td>
        </tr>
        {{ end }}
      </tbody>
    </table>
  </body>
</html>
{{ end }}
```

Load the template file and reference its name from the email receiver's `html` field:

```yaml
global:
  smtp_smarthost: 'smtp.example.org:587'
  smtp_from: 'alertmanager@example.org'

route:
  receiver: 'email-notifications'

receivers:
- name: 'email-notifications'
  email_configs:
  - to: 'oncall@example.org'
    html: '{{ template "email.myorg.html" . }}'

templates:
- '/etc/alertmanager/templates/*.tmpl'
```

See the [`<email_config>` reference](configuration.md#email_config) for SMTP authentication and TLS options.

## Sending notifications to Gotify with a custom webhook payload

[Gotify](https://gotify.net) doesn't have a dedicated Alertmanager receiver, but its push message API is a simple JSON POST, so it can be driven directly with the generic [`webhook_config`](configuration.md#webhook_config) and its `payload` field.

```yaml
receivers:
- name: gotify
  webhook_configs:
  - url: 'https://gotify.example.com/message'
    http_config:
      authorization:
        credentials_file: /etc/alertmanager/secrets/gotify-secret/token
    payload:
      title: '{{ .Status | toUpper }} {{ .CommonLabels.alertname }}'
      priority: '{{ if eq .Status "firing" }}5{{ else }}0{{ end }}'
      message: |
        {{ range .Alerts }}{{ .Annotations.summary }}
        {{ end }}
```

A more complete version of this example, with Markdown formatting and separate sections for firing and resolved alerts, is available in [`examples/webhook/gotify.yml`](https://github.com/prometheus/alertmanager/blob/main/examples/webhook/gotify.yml).
