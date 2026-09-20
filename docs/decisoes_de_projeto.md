# 📘 Decisões de Projeto — CineData Analytics

Documento técnico com as principais decisões que tomei ao longo do desenvolvimento e os trade-offs que considerei.

---

## 1. Decisões Arquiteturais

Organizei o projeto em quatro schemas dentro do catalog `cinedata_medallion`: `landing`, `bronze`, `silver` e `gold`. O `landing` abriga um Volume chamado `inputs`, onde ficam os cinco CSVs brutos. Escolhi essa separação porque ela mantém distinção clara entre arquivos brutos e tabelas processadas, está alinhada com a Arquitetura Medalhão e permite aplicar políticas de acesso por camada no Unity Catalog. A nomenclatura segue `<catalog>.<camada>.<tabela>`, tudo em snake_case.

Para as surrogate keys da Gold, escolhi `row_number().over(Window.orderBy(...))`. Porque o custo de performance é aceitável no nosso volume e a legibilidade vale muito no dia a dia.

Para auditoria, usei nomes diferentes por camada: `ingestion_timestamp` na Bronze e `processed_timestamp` na Silver. Isso evita confusão sobre o significado de cada coluna. A Gold não tem coluna de auditoria, porque o grão é de negócio e o timestamp não agrega valor analítico.

Quanto ao modo de escrita, a Bronze usa `append` para manter histórico imutável de ingestões, enquanto Silver e Gold usam `overwrite` porque representam estado atual derivado de forma determinística.

---

## 2. Decisões da Camada Bronze

Li os CSVs de forma pura, com `header=True` e `inferSchema=True`, sem transformação de negócio. O princípio da Bronze é preservar os dados como chegaram. A única adição foi a coluna `ingestion_timestamp`, gerada com `current_timestamp()` na cadeia que termina em `.write()`.

Para a `tb_cotacao_dolar`, consumi o endpoint PTAX OData do Banco Central, parametrizado por dois widgets (`data_inicio` e `data_fim`) no formato `MM-DD-AAAA`, com fallback para os últimos 7 dias corridos. Pedi 7 dias em vez de 1 porque a API não retorna cotação em fins de semana e feriados — assim garanto pelo menos 5 dias úteis.

---

## 3. Decisões da Camada Silver

Na `tb_info_filmes`, dedupliquei mantendo a versão mais recente por `ingestion_timestamp`. Normalizei o status em três etapas (trim + lower, remoção de hífens, tradução para PT com fallback `"Não Informado"`). Para as datas, testei três formatos em ordem: ISO, americano e brasileiro. Escolhi essa ordem porque o ISO é o mais universal e datas ambíguas como `05/06/2018` caem no americano. O runtime zero vira NULL porque filme sem duração é inválido. Também implementei uma limpeza de sinopse (remoção de barras invertidas, colapso de aspas múltiplas e remoção da tagline colada), que sobrescreve a coluna `sinopse` antes do select final.

Na `tb_financeiro_filmes`, limpei `$`, `USD` e o sufixo `M` (que multiplica por 1.000.000). Negativos e zeros viram NULL. Converti para BRL usando a cotação mais recente, porque só tenho 7 dias de cotação e não posso usar a taxa da época de cada filme. O lucro é `receita - orçamento`, e a margem é calculada só quando a receita é positiva, para evitar divisão por zero.

Na `tb_metricas_engajamento`, substituí vírgula decimal por ponto e apliquei `try_cast`. Adicionei um teto de 1000 para a popularidade — isso resolve tanto os anos que vazaram quanto valores absurdos. Notas fora de 0–10 viram NULL, conforme o enunciado pede ("desconsiderar", não "corrigir dividindo por 10").

Na `tb_avaliacoes_usuarios`, dedupliquei registros idênticos em `(id, nome, nota, comentario)`. Notas fora de 0–10 viram NULL. Comentários vazios viram `"Sem comentário"`.

Na `tb_generos`, fiz split por `[,;|]` (os três separadores presentes na origem), explode e limpeza. A decisão mais importante aqui foi usar whitelist com os 19 gêneros oficiais do TMDB. Como gênero é um domínio fechado, whitelist remove 100% do lixo de uma vez.

Na `tb_pessoas_empresas`, unifiquei quatro colunas (`cast`, `directors`, `writers`, `production_companies`) numa estrutura com `tipo_entidade`. Apliquei blacklists de placeholders e de idiomas/países, além de heurísticas para distinguir nomes próprios de keywords. Também implementei filtros específicos para resíduos do column shift: rejeitar strings terminando em extensões de imagem (`.jpg`, `.png`, `.gif`, `.webp`, `.svg`), strings começando com `/`, strings começando com caracteres especiais (`#`, `&`, `@`, `%`, `*`) e parênteses desbalanceados. Mesmo assim, ainda há poluição residual.

Na `tb_cotacao_dolar`, consolidei a última cotação de cada dia, gerei um calendário contínuo com `sequence()` e apliquei Forward Fill com `last(ignorenulls=True)`. Isso garante que fins de semana e feriados herdem a cotação do último dia útil.

---

## 4. Decisões da Camada Gold

Modelei a Gold como Star Schema, com cinco dimensões, uma tabela fato e três bridge tables. Usei bridges porque filme tem N:N com gêneros, pessoas e produtoras — sem elas, o grão da fato seria duplicado. A fato foi construída a partir da `dim_movies` com LEFT JOINs, garantindo que todo filme apareça, mesmo sem métricas. Mantive o typo `blr` (com L) do enunciado, para aderência máxima.

Na `gold_genai_movies_context`, o principal desafio foi a armadilha dos nulos. `concat()` retorna NULL se qualquer campo for nulo, então apliquei `coalesce()` em cada campo com fallbacks específicos, incluindo o caso do diretor `"English"` (que veio de column shift) sendo tratado como `"diretor não informado"`.

Para o recorte temporal das queries 5 e 6, calculei a data limite como o lançamento mais recente com `status_filme = 'Lançado'` e `data_lancamento <= CURRENT_DATE`. Usei essa data como referência para os cortes de 2 e 5 anos.

---

## 5. Trade-offs Conscientes

**Column shift — filtrar vs. consertar.** Optei por filtrar, não consertar. O enunciado pede "remover resíduos" (não reconstruir a origem), consertar seria complexo (cada linha tem deslocamento diferente) e a perda de dados sujos é aceitável em favor da qualidade.

**Cotação do dólar.** Usei a cotação mais recente para todos os filmes, mesmo sendo economicamente impreciso, porque só tenho 7 dias de cotação disponíveis.

**Granularidade da dim_people.** Uma linha por `(nome_pessoa, tipo_pessoa)`. Se a mesma pessoa é Ator e Diretor, aparece duas vezes — porque o papel define a identidade no contexto do cinema.

**Timestamp na Silver.** Adicionei `processed_timestamp` mesmo sem a atividade exigir. Custo zero, linhagem completa.