# {{ .Title }}
{{ with .Description }}
> {{ . }}
{{ end }}
Source: {{ .Permalink }}
{{ if .Translations }}{{ range .Translations }}Translation ({{ .Language.LanguageName }}): {{ .Permalink }}
{{ end }}{{ end }}
{{ .RawContent }}
