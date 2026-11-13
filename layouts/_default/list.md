# {{ .Title }}
{{ with .Description }}
> {{ . }}
{{ end }}
Source: {{ .Permalink }}

{{ range .Pages.ByWeight }}- [{{ .Title }}]({{ with .OutputFormats.Get "markdown" }}{{ .Permalink }}{{ else }}{{ .Permalink }}{{ end }}){{ with .Description }} — {{ . }}{{ end }}
{{ end }}
