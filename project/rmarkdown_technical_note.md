# Reusable R Markdown techniques

This note summarizes the document and R-chunk techniques used in
`Version_Control.Rmd` that can be adapted to this project. The analysis topic
and thesis-specific data are intentionally not reproduced here.

## R Markdown document features

- **YAML front matter** configures the title, authors, date, output formats,
  document options, bibliography, and LaTeX header settings. A document can
  declare `html_document` and `pdf_document` outputs; choose the formats that
  suit the report. PDF output requires a working LaTeX installation.
- **Generated values in prose** use inline R, for example
  `` `r format(Sys.Date(), "%d %B %Y")` `` for a render-date stamp. This keeps
  reported calculations in sync with the code.
- **Structure and navigation** use Markdown headings, a table of contents,
  page breaks, and footnotes. Heading attributes can mark sections unnumbered
  or omit them from the table of contents.
- **Citations** use a BibTeX bibliography file and Pandoc citation keys, for
  example `[@reference-key]`. Keep the bibliography file alongside the
  document or use a deliberate relative path.
- **Figures** can have captions, alignment, and output sizing set in chunk
  options. `fig.show='hold'` places multiple plots from a chunk together.
- **Tables** can be produced from R and formatted for the target output. Check
  that table and plot output is compatible with each declared format.

## Chunk patterns

A hidden setup chunk is a good place for document-wide knitr defaults and
shared package imports:

```{r setup, include=FALSE}
knitr::opts_chunk$set(
  echo = FALSE,
  warning = FALSE,
  message = FALSE,
  fig.align = "center",
  out.width = "100%"
)
```

Use per-chunk options to override defaults where needed:

```{r example-figure, echo=TRUE, fig.cap="Example caption", out.width="75%"}
plot(1:5)
```

Useful options demonstrated in the source include `include=FALSE` for setup
code that should run without appearing, `echo=FALSE` to hide code while
showing results, `warning=FALSE` and `message=FALSE` to suppress chunk
diagnostics in the rendered report, and `fig.cap`, `fig.align`, `out.width`,
and `fig.show` to control figures. Suppress diagnostics only when they are
not important to readers; otherwise leave them visible while developing.

Chunk options can depend on render parameters, such as `echo = params$Details`,
to show detailed code selectively. Declare parameters in the YAML `params:`
section when using this pattern, and provide their values when rendering.

## Packages used in R chunks

The source loads these packages in its setup chunk. Their roles below are
based on the setup comments and functions used there; select only the packages
needed for a particular report.

| Purpose | Packages |
|---|---|
| Data import and transformation | `readxl` (Excel import), `dplyr`, `tidyr`, `tibble`, `magrittr`, `tidyverse` |
| Text and factor handling | `stringr`, `forcats`, `plyr` |
| Static plots and plot composition | `ggplot2`, `viridis`, `GGally`, `ggridges`, `ggpol`, `patchwork` |
| Interactive visualizations | `plotly`, `networkD3` |
| Tables and document rendering | `knitr`, `kableExtra`, `rmarkdown` |
| Modeling and model visualization | `caret`, `rpart.plot` |
| Capturing web graphics for documents | `webshot` |

`tidyverse` already includes several listed data tools, but the source also
loads some components explicitly. Avoid loading both unless the explicit
imports help make dependencies clear. Interactive widgets such as Plotly or
networkD3 are most useful in HTML output; PDF output may require a static
fallback or an image capture workflow such as the `webshot` approach used in
the source.

## Practical adaptations for this project

- Prefer project-relative input and output paths over machine-specific
  absolute paths, so the document can be rendered from another computer.
- Keep package setup, data preparation, analysis, and presentation in clearly
  named chunks.
- Render after editing to catch missing packages, path issues, and format
  incompatibilities. Set chunk output options to improve presentation, not to
  conceal errors during development.
- Use inline R for numbers that should update automatically, and generated
  tables/figures rather than manually transcribed results.

## Parameterized reports

`Version_Control.Rmd` uses `echo = params$Details` on long modelling chunks
but the YAML header of that file has no `params:` block, so rendering it as is
would fail. Always declare the parameter:

```yaml
params:
  Details: false
```

Render with another value without editing the file:

```r
rmarkdown::render("report.Rmd", params = list(Details = TRUE))
```

Tested in this repository's R 4.6.1: both values render correctly in HTML.

## Cross-references and captions

- **Section anchors**: attach `{#my-id}` to a heading and link with
  `[text](#my-id)`. This works in HTML and PDF output, and is what the source
  uses for navigation between sections.
- **Figure captions** come from `fig.cap` in the chunk header; with
  `bookdown` (not used in the source) they can also be referenced
  automatically.
- **Table captions** are set with `kable(..., caption = "...")`.
- **Footnotes**: `text[^id]` in the prose and `[^id]:` definition below the
  paragraph. Footnotes may contain LaTeX math such as `$\chi^2$`.
- **Unnumbered headings**: `# Title {.unlisted .unnumbered}` (source uses it
  for the acknowledgements and abstract pages).

## Tables with `kable` and `kableExtra`

The source builds almost every table with this pipeline. Replace
`format = "latex"` by omitting `format` (or `"html"`) for HTML output, or leave
it unset so knitr picks the format automatically.

```r
df %>%
  kable(align = "lrrr", booktabs = TRUE, caption = "Caption") %>%
  kable_styling(latex_options = c("hold_position", "scale_down")) %>%
  add_header_above(c(" " = 1, "Group A" = 2, "Group B" = 2)) %>%
  pack_rows("Subset 1", 1, 3) %>%
  row_spec(0, bold = TRUE) %>%
  row_spec(c(1, 4), bold = TRUE, color = "white", background = "#F2637F") %>%
  row_spec(2, hline_after = TRUE) %>%
  column_spec(1, width = "10em") %>%
  column_spec(3, border_right = TRUE)
```

What each helper does:

| Function | Use |
|---|---|
| `kable(align = ...)` | per-column alignment, e.g. `"llrrr"` |
| `kable_styling(latex_options = "hold_position")` | keep the table where it is written |
| `kable_styling(latex_options = "scale_down")` | shrink a wide table to the page width |
| `kable_styling(full_width = TRUE)` | stretch to the text width |
| `add_header_above()` | grouped column headers |
| `pack_rows()` | labelled row groups |
| `row_spec()` / `column_spec()` | bold, colours, widths, borders |

A common trick: build the table with `t()`, `as.data.frame()` and
`magrittr::set_colnames()` first, then pass it to `kable`.

## Figure layout

- **Side by side**: put two plots in one chunk with
  `out.width = "49%", fig.show = "hold"`. Each plot keeps its own code, and the
  chunk caption applies to the pair.
- **One composite figure**: `patchwork` (`p1 + p2`, `p1 / p2`) gives one
  aligned plot with a single caption. Prefer it when the plots share an axis.
- **Consistent colours**: define a palette once (the source uses
  `viridis::magma()` and a helper to pick a highlight colour) and reuse it with
  `scale_fill_manual()`, so that categories keep the same colour across plots.
- **Reusing a plot recipe**: wrap the `ggplot` call in a function that
  returns the plot, then call it with different data or filters.

## Reusable functions in chunks

The source defines helper functions in a dedicated chunk (often with
`echo = FALSE`) and calls them from later chunks. This pattern keeps the
report short and avoids copy-paste when the same analysis is repeated on
subsets (for example `Model.Exam(Data = subset)`). Remember to call
`set.seed()` in the chunk that runs a random procedure so that figures stay
reproducible between renders.

## Modelling snippets (`caret` and `rpart`)

The source fits a decision tree with a train/test split and 10-fold
cross-validation. The same structure fits the cross-validation requirement of
this project (GLM/GAM feature selection) with another `method`.

```r
set.seed(1234)
idx      <- createDataPartition(df$y, p = 0.8, list = FALSE)
df.train <- df[idx, ]
df.test  <- df[-idx, ]

ctrl <- trainControl(method = "cv", number = 10)
fit  <- train(y ~ ., data = df.train, method = "rpart",
              trControl = ctrl, metric = "Kappa", tuneLength = 30)

rpart.plot(fit$finalModel, type = 0, fallen.leaves = TRUE)
```

- `createDataPartition()` keeps the class balance in both subsets.
- `metric = "Kappa"` is preferred over accuracy for unbalanced outcomes.
- Variable importance is available from `fit$finalModel$variable.importance`.
- Other `caret` methods (for example `"glm"`) reuse the same `trainControl`.

## Interactive graphics and PDF output

- `plotly` and `networkD3` produce HTML widgets and only display in HTML
  output.
- For PDF output, the source keeps a commented recipe: save the widget with
  `htmlwidgets::saveWidget()`, then capture it as an image with
  `webshot::webshot()` and include the PNG. The `webshot` package needs
  PhantomJS; its successor `webshot2` uses Chrome and installed correctly here.
- `always_allow_html: yes` in the YAML is required when widgets or HTML tables
  are used with PDF output.

## LaTeX details for PDF output

- `header-includes` with `\usepackage[french]{babel}` sets hyphenation and
  French typography; use the matching language for the project.
- Page control in the source: `\newpage`, `\thispagestyle{empty}`,
  `\pagenumbering{arabic}`, `\tableofcontents`, and `\newline` to end a
  paragraph with a line break.
- Layout: `geometry: "left=4cm, right=3cm, top=2.5cm, bottom=2.5cm"` and
  `fontsize: 11pt`.
- `\rule{\textwidth}{1pt}` draws a horizontal line to separate major parts.
- `kable(booktabs = TRUE)` needs the `booktabs` LaTeX package, which is
  included in common TeX distributions.

## Verification notes

- The table, figure, `patchwork`, `caret`/`rpart` and `plotly` snippets above
  were rendered to HTML with R 4.6.1 (`kableExtra`, `rpart.plot`, `plotly`
  and `webshot2` were installed for the test).
- PDF output was not tested because no LaTeX (`pdflatex`) is on the PATH on
  this machine. LaTeX-specific options (`latex_options`, `babel`, `geometry`)
  are described from the source file only.
- The source loads its data from absolute paths on another computer; this
  project should use project-relative paths such as `../data/mtpl_be.rda`.
