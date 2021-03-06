*This script generates the plots used in the article.*

``` r
library("magrittr", warn.conflicts = FALSE)
library("tidyr", warn.conflicts = FALSE)
library("dplyr", warn.conflicts = FALSE)
library("stringr")
library("readr")
library("broom")
library("grid")
library("ggplot2")
source("R/00_utils_formatting.R")
inv_logit <- gtools::inv.logit

set_condition <- . %>% 
  factor(., levels = c("facilitating", "neutral"))

# Shortcut so that column widths are mapped to appropriate mm values
ggsave_cols <- function(cols = 1, ...) {
  width <- c(`1` = 90, `1.5` = 140, `2` = 190)[as.character(cols)]
  ggsave(..., width = width, units = "mm", dpi = 600)
}
```

Figure 1
--------

``` r
prep_formant_csv <- function(path) {
  df <- read_csv(path, na = "--undefined--")

  # Clean up column names
  names(df) %<>% str_replace(".Hz.", "") %>%
    str_replace(".s.", "") %>%
    str_replace("time", "Time")

  # Extract sound from filename
  sound <- path %>% 
    str_extract("(?<=the_).(?=\\.csv)") %>% 
    tolower

  # Drop unnecessary columns (like bandwidth)
  df %<>% select(Time, F1, F2) %>%
    na.omit %>%
    mutate(Sound = sound)

  df
}

# Load and reduce the formant files
formants <- list.files("phonetics/formants/", full.names = TRUE) %>% 
  lapply(prep_formant_csv) %>% 
  bind_rows %>% 
  # Convert to long format (so there's a Formant-Name column)
  gather(Formant, Hz, -Time, -Sound) %>%
  mutate(ms = Time * 1000) %>%
  filter(150 <= ms, ms <= 405)

# Convert sound name into "Token"
formants$Token <- formants$Sound %>%
  str_replace("^v$", "neut.") %>%
  str_replace("^(d|b)$", "/\\1/ faci.") %>%
  factor(levels = c("neut.", "/d/ faci.", "/b/ faci."))

formant_inset_legend <- theme(
  legend.position = c(.5, 0),
  legend.justification = c(.5, 0),
  legend.background = element_rect(fill = "white", colour = "grey"),
  legend.key.size = unit(4, "mm"),
  legend.direction = "horizontal",
  plot.margin = unit(rep(2, 4), "mm")
)


p_base <- ggplot(data = formants) +
  aes(x = ms, y = Hz / 1000, color = Token, shape = Token) +
  labs(x = "Time (ms)", y = "F1 and F2 frequencies (kHz)") +
  ylim(0, NA) +
  theme_bw(base_size = 10) +
  formant_inset_legend
  
p <- p_base + geom_point() + scale_color_brewer(palette = "Dark2")
p
```

<img src="plots_files/figure-markdown_github/unnamed-chunk-1-1.png" title="" alt="" style="display: block; margin: auto;" />

*Figure 1.* F1 and F2 formants of the tokens of the determiner *the* for the three item types. Note the canonical formant transitions for the facilitating tokens and the steady formant values in the neutral token.

``` r
ggsave_cols("plots/fig1.png", p, cols = 1, height = 80)
ggsave_cols("plots/fig1.eps", p, cols = 1, height = 80, device = cairo_ps)
```

![](plots_files/figure-markdown_github/unnamed-chunk-2-1.png)<!-- -->

``` r
p_bw <- p_base + geom_point(alpha = .75) + aes(color = NULL)
ggsave_cols("plots/bw_fig1.png", p_bw, cols = 1, height = 80)
ggsave_cols("plots/bw_fig1.eps", p_bw, cols = 1, height = 80, device = cairo_ps)
```

Eyetracking figures
===================

``` r
# x axis limits and landmarks
set_xlims <- coord_cartesian(xlim = c(-650, 1300))
landmarks <- geom_vline(xintercept = c(-600, 0, 850), linetype = "dashed", 
                        alpha = .5)

x_gaze_lab <- labs(x = "Time since target-word onset (ms)")
y_gaze_lab <- labs(y = "Proportion looking to target")

inset_legend <- theme(
  legend.position = c(0.015, 1),
  legend.justification = c(0, 1),
  legend.background = element_rect(fill = "white", color = "grey"),
  plot.margin = unit(rep(2, 4), "mm")
)

theme_big <- theme_bw(base_size = 12) + inset_legend
theme_small <- theme_bw(base_size = 10) + inset_legend
```

Figure 2
--------

``` r
set_condition <- . %>% 
  factor(., levels = c("facilitating", "neutral"))

# Full looking data
looks_full <- read_csv("data/binned_looks.csv") %>%
  filter(Condition != "filler", -600 <= Time, Time < 1300) %>%
  mutate(Condition = set_condition(Condition))

p_base <- 
  ggplot(looks_full) +
  aes(x = Time, y = Proportion, color = Condition, shape = Condition) +
  landmarks + set_xlims +
  scale_color_brewer(palette = "Set1") +
  x_gaze_lab + y_gaze_lab

p2 <- p_base + 
  stat_summary(fun.data = "mean_se", geom = "pointrange", size = .9) + 
  theme_big
p2
```

<img src="plots_files/figure-markdown_github/unnamed-chunk-3-1.png" title="" alt="" style="display: block; margin: auto;" />

*Figure 2.* Proportion looking to target from onset of *the* to 1250 ms after target-word onset in the two conditions. Symbols and error bars represent observed means Â±SE. Dashed vertical lines mark onset of *the*, target-word onset, and target-word offset.

``` r
ggsave_cols("plots/fig2.png", p2, cols = 2, height = 110)
ggsave_cols("plots/fig2.eps", p2, cols = 2, height = 110, device = cairo_ps)
```

![](plots_files/figure-markdown_github/unnamed-chunk-4-1.png)<!-- -->

``` r
p2_small <- p_base + 
  stat_summary(fun.data = "mean_se", geom = "pointrange") + 
  theme_small
ggsave_cols("plots/fig2_small.png", p2_small, cols = 1.5, height = 80)


p2_bw <- p2 + aes(color = NULL)
ggsave_cols("plots/bw_fig2.png", p2_bw, cols = 2, height = 110)
ggsave_cols("plots/bw_fig2.eps", p2_bw, cols = 2, height = 110, device = cairo_ps)
```

Figure 3
--------

``` r
## Get predicted values and standard errors from the model
library("AICcmodavg")
load("data/models.Rdata")
model <- models$main$m_cond_ranef
model_data <- read_csv("data/model_data.csv") %>%  
  mutate(Condition = set_condition(Condition))

# Create a design data-frame (i.e., times and conditions, no participants)
df_design <- model_data %>% 
  select(Bin, Time, ot1:ot3, Condition) %>% 
  distinct

# Including columns with unmodeled values breaks predictSE, so drop those.
df_new <- df_design %>% select(-Bin, -Time)

model_fits <- predictSE(model, df_new) %>% 
  as_data_frame %>% 
  bind_cols(df_design) %>% 
  rename(se = se.fit) %>% 
  mutate(ymin = fit - se, ymax = fit + se)

p3_base <- ggplot(model_fits) + 
  aes(x = Time, y = fit, ymax = ymax, ymin = ymin, 
      shape = Condition, fill = Condition, color = Condition) + 
  geom_ribbon(alpha = .2, color = NA) + 
  x_gaze_lab + 
  labs(y = "Emp. logit of looking to target") +
  scale_color_brewer(palette = "Set1") + 
  scale_fill_brewer(palette = "Set1")

p3 <- p3_base + 
  geom_point(size = 3.5) + 
  geom_line(size = .9) + 
  theme_big

# unit conversion for caption
elogits <- 0:3 / 2
props <-  plogis(elogits) %>% round(2) %>% 
  remove_leading_zero %>% 
  paste0(collapse = ", ")

p3
```

<img src="plots_files/figure-markdown_github/unnamed-chunk-5-1.png" title="" alt="" style="display: block; margin: auto;" />

*Figure 3.* Growth curve estimates of looking probability during analysis window. Symbols and lines represent model estimates, and ribbon represents Â±SE. Empirical logit values on y-axis correspond to proportions of .5, .62, .73, .82. Note that the curves are essentially phase-shifted by 100 ms.

``` r
# Smaller version
p3_smaller <- p3_base + 
  geom_point() + 
  geom_line() + 
  theme_small

ggsave_cols("plots/fig3.png", p3_smaller, cols = 1, height = 90)
ggsave_cols("plots/fig3.eps", p3_smaller, cols = 1, height = 90, 
            device = cairo_ps)
```

![](plots_files/figure-markdown_github/unnamed-chunk-6-1.png)<!-- -->

``` r
# Smaller version
p3_bw_smaller <- p3_base + 
  aes(fill = NULL, color = NULL, linetype = Condition) + 
  geom_point() + 
  geom_line() + 
  theme_small

ggsave_cols("plots/bw_fig3.png", p3_bw_smaller, cols = 1, height = 90)
ggsave_cols("plots/bw_fig3.eps", p3_bw_smaller, cols = 1, height = 90, 
            device = cairo_ps)
```
