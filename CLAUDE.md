# Commodities Earnings Monitor

Painel de acompanhamento de earnings/eventos relevantes para empresas de commodities (petróleo, cobre, alumínio, ouro, platina/paládio), voltado para uso de investidor profissional. Toda a informação vive em um único arquivo HTML autocontido: `index.html`.

## Estrutura do arquivo

`index.html` é uma página estática com um JSON embutido em:
```html
<script id="data" type="application/json">{...}</script>
```

Schema do JSON:
- `data.companies[]` — `{id, ticker, name, commodities[], status}`. `status` ∈ `clear | new | needs_verification | not_traded`.
- `data.filings{}` — chaveado por `company_id`, um objeto por empresa com análise profunda:
  ```
  {
    company_id, accession_number, document_type, document_type_label,
    sources: [{filename, label, source_url, captured_at}],
    tabs: { earnings_summary, conference_call_summary, commodity_commentary },
    commentary_sources: [{outlet, title, url, captured_at}],
    generated_by, generated_at, note_for_reviewer
  }
  ```
- `data.checked_at` — timestamp ISO da última varredura de atualidade (freshness check).

Ordem das abas na UI (`renderFiling` no `<script>` da página): **comodity_commentary primeiro (ativa por padrão), earnings_summary segundo, conference_call_summary terceiro.** Isso é proposital — feedback explícito do usuário: comentário de mercado é mais importante que o dado financeiro cru.

## Regras de sourcing (não negociáveis)

1. **Dados financeiros (`earnings_summary`, `conference_call_summary`, `sources[]`)**: SOMENTE de fontes primárias oficiais — SEC EDGAR (10-Q, 10-K, 8-K, 6-K), site de Relações com Investidores da própria empresa, transcrições de conference call, LSE RNS, JSE SENS, HKEX, CVM, ASX. Nunca estimar, inferir ou arredondar números que não estejam explicitamente no documento primário.
2. **Comentário de mercado (`commodity_commentary`, `commentary_sources[]`)**: matéria de imprensa financeira ESPECIFICAMENTE sobre o resultado/earnings da empresa — Bloomberg em primeiro lugar, depois Reuters, FT, WSJ, ou fallback regional confiável (Valor, InfoMoney, The Edge Malaysia, Mining Weekly, Mining.com, etc. quando cabível). Não vale notícia genérica sobre a empresa que não seja sobre o resultado/evento em questão.
3. **Nunca fabricar URL, citação, número ou reação de mercado.** Se não encontrar uma fonte de imprensa boa o bastante, isso deve ser dito explicitamente no `note_for_reviewer` — não preencher com nada aproximado.
4. **Regra de atualidade (crítica, em vigor desde 07/07/2026)**: toda matéria de imprensa citada em `commentary_sources` precisa ser datada do dia da atualização em diante — nunca reaproveitar notícia antiga como se fosse novidade atual. Se checar uma empresa e a notícia mais recente e relevante ainda for antiga, é melhor deixar como está (ou marcar `note_for_reviewer`) do que citar algo desatualizado como se fosse novo.
5. **Padrão de qualidade = manchete no estilo "Oil Major Shell Cashes In On Volatile Energy Markets"**: específica sobre o earnings, de veículo confiável, datada corretamente.

## Padrão de exatidão (usuário é investidor profissional)

O usuário usa este painel para decisões de investimento reais. Prioridade é completude e precisão sobre velocidade:
- Checar múltiplas fontes antes de confirmar um evento como novo/relevante.
- Nunca omitir silenciosamente uma lacuna — sinalizar no `note_for_reviewer`.
- Cuidado especial com eventos parecidos mas distintos (ex.: distinguir a ASSINATURA de um acordo de M&A da sua CONCLUSÃO/fechamento meses depois — já ocorreu confusão desse tipo com Anglo American/Codelco).

## Fluxo de atualização

1. Editar o JSON embutido em `index.html` (via script — não editar manualmente strings JSON grandes à mão; usar Python `re.search` no bloco `<script id="data">`, `json.loads`, mutar, `json.dumps(..., ensure_ascii=False)`, reinjetar).
2. Atualizar `data.checked_at` com o timestamp ISO da varredura.
3. Commit + push do `index.html` no repositório.
4. Para lote de várias empresas: pesquisar cada uma separadamente (pode paralelizar via sub-agentes de pesquisa, MAS cada sub-agente deve **retornar apenas um JSON estruturado** — nunca deixar um sub-agente escrever direto no arquivo ou publicar nada sozinho, para evitar condição de corrida. A thread principal sempre faz o merge e o commit.

## O que NÃO fazer

- Não estimar datas de earnings — só usar datas oficialmente confirmadas (calendário de IR, comunicado da empresa).
- Não deixar um sub-agente de pesquisa usar a ferramenta de publicação/deploy sozinho.
- Não reescrever tudo do zero a cada atualização — só tocar nas empresas que genuinamente têm novidade (evitar "churn" desnecessário).
