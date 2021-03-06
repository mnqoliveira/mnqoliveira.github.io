My Kindle Activity
================

-   [Preamble](#preamble)
-   [For how long has Frodo kept me
    company?](#for-how-long-has-frodo-kept-me-company)
-   [When was I reading what?](#when-was-i-reading-what)
-   [Time to go to bed](#time-to-go-to-bed)

### Preamble

I find Kindle Insights not as insightful as I wanted them to be, so I
requested my Kindle data and dug into some of my reading habits by
myself. This analysis uses data from the ReadingSession file (with some
personal data removed).

``` r
library("tidyverse")
library("lubridate")

library("paletteer")

# For the failed webscrapping part
#library('rvest')
```

``` r
read_session <- read.csv("./data/Kindle.Devices.ReadingSession_mod.csv") %>%
  filter(ASIN != "")
```

I soon found out the logs do not include the name of the books, but they
refer to an ASIN, which is a code for Amazon products. Well, I could
identify to which product they corresponded by using the website. It
should be easy to retrieve this information.

It should. But I naively forgot that this would likely not be allowed so
here I am with a chunk of code I do not recommend you to run or Amazon
will flag you and block you from running automated searches.

``` r
# asins <- data.frame(ASIN=unique(read_session$ASIN),
#                     title=NA)
# 
# for (asin_it in 1:nrow(asins)){
# 
#   asin <- asins[asin_it, "ASIN"]
#   page_url <- paste0('https://www.amazon.com/gp/product/', asin)
#   possibleError <- tryCatch(
#     webpage <- read_html(page_url),
#     error=function(e) e
#   )
# 
#   print(possibleError)
# 
#   if(!inherits(possibleError, "error")){
# 
#     webpage <- read_html(page_url)
# 
#     webpage_ <- webpage %>%
#       html_node("div") %>%
#       html_nodes("meta") %>%
#       html_attrs() %>%
#       do.call(c, .)
# 
#     title <- webpage_[grep("title", webpage_) + 1]
#     names(title) <- NULL
# 
#     if (!is.null(title)){
#       asins$title[asin_it] <- title
#     }
# 
#     print(title)
# 
#   }
# 
# }
# 
# list_str <- "Amazon.com|Amazon.com: |Kindle edition.+|: Kindle Store| eBook :.+"
# list_str2 <- " .Portuguese Edition.| .French Edition.| .Spanish Edition."
# 
# asins_ <- asins %>%
#   mutate(title = gsub(list_str, "", title)) %>%
#   mutate(title = gsub(list_str2, "", title)) %>%
#   mutate(title = gsub(" - $", "", title))
```

After briefly succeeding, i. e., the code worked for a few IDs and
stopped, I realized I would not be able to pursue this path. So I
decided to do gather the names manually. I did not read that many books
after all.

``` r
# asins <- data.frame(ASIN = unique(read_session$ASIN)) %>%
#   mutate(path_book = paste0('https://www.amazon.com/gp/product/', ASIN))
# 
# write.csv(asins, file="./data/asins.csv", row.names = FALSE)
```

This version includes the names that I manually retrieved from the
website.

``` r
asins_ <- read.csv("./data/asins_mod.csv")

read_session_ <- read_session %>%
  left_join(asins_) %>%
  select(-contains("marketplace"), -contains("content"), -path_book)
```

``` r
head(read_session_)
```

    ##        start_timestamp        end_timestamp       ASIN total_reading_millis
    ## 1 2018-01-20T16:47:58Z 2018-01-20T16:50:52Z B01B173GA6               173700
    ## 2 2018-01-22T01:43:28Z 2018-01-22T02:10:26Z B00BWJC5F6              1618100
    ## 3 2017-12-29T01:06:06Z 2017-12-29T01:11:47Z B00849BWNS               340500
    ## 4 2017-12-30T02:14:13Z 2017-12-30T02:14:17Z B00849BWNS                 3100
    ## 5 2018-01-20T04:15:37Z 2018-01-20T04:20:23Z B01B173GA6               286100
    ## 6 2018-01-07T01:35:22Z 2018-01-07T01:38:13Z B00849BWNS               171100
    ##   number_of_page_flips               title
    ## 1                    1 Pride and Prejudice
    ## 2                   61  The Princess Bride
    ## 3                   20     Utilitarianism 
    ## 4                    1     Utilitarianism 
    ## 5                    9 Pride and Prejudice
    ## 6                    6     Utilitarianism

### For how long has Frodo kept me company?

As I was anxious because of my PhD project, I often went back to my
comfort books, which were mainly The Lord of the Rings, after I bought
it, and Pride and Prejudice, before that.

``` r
read_session_sum <- read_session_ %>%
  group_by(title, ASIN) %>%
  summarise(tot_hours = sum(total_reading_millis, na.rm = TRUE) / (1000 * 3600)) %>%
  ungroup() %>%
  arrange(desc(tot_hours)) %>%
  filter(tot_hours > 5) %>%
  mutate(title2 = factor(title,
                         levels = title,
                         labels = title,
                         ordered = TRUE))

ggplot(read_session_sum) +
  geom_col(aes(y=title2, x=tot_hours)) +
  labs(x = "Total hours", y = "Book") +
  theme(legend.position = "bottom",
        panel.background = element_rect(fill = "gray95"),
        axis.text.x = element_text(angle = 90,
                                   hjust = 1, 
                                   size = 8),
        legend.text = element_text(size = 8))
```

<img src="notebook_files/figure-gfm/fig1-1.png" style="display: block; margin: auto;" />

### When was I reading what?

I was also curious regarding at what moment in recent years I was
reading, well, LotR, but also the other books from my top 15. The plot
also pointed to how often daily reading didn???t sum up to more than one
hour.

``` r
topbooks <- read_session_sum[1:15, "title"]

appear_order <- read_session_ %>%
  mutate(total_reading_millis = total_reading_millis / (1000 * 60),
         start_timestamp = gsub("Z", "", start_timestamp),
         start_timestamp = ymd_hms(start_timestamp)) %>%
  group_by(title) %>%
  summarise(date_ = min(start_timestamp, na.rm = TRUE)) %>%
  ungroup() %>%
  arrange(date_) %>%
  mutate(order_ = row_number()) %>%
  select(-date_)

read_session_all <- read_session_ %>%
  select(start_timestamp, ASIN, total_reading_millis, title) %>%
  filter(title %in% unlist(topbooks)) %>%
  mutate(total_reading_millis = total_reading_millis / (1000 * 60),
         start_timestamp = gsub("Z", "", start_timestamp),
         start_timestamp = ymd_hms(start_timestamp),
         date_ = date(start_timestamp)) %>%
  group_by(date_, title) %>%
  summarise(total_time = sum(total_reading_millis)) %>%
  ungroup() %>%
  left_join(appear_order) %>%
  arrange(order_) %>%
  mutate(title2 = factor(title,
                         levels = title,
                         labels = title,
                         ordered = TRUE))

ggplot(read_session_all) +
  geom_point(aes(x=date_, y=total_time, 
                 col=title2),
             position="jitter") +
  labs(x = "Date", y = "Daily duration [minutes]", color="") + 
  scale_discrete_manual(values = paletteer_d("ggsci::category20_d3"),
                        aesthetics = "colour") +
  scale_y_continuous(breaks=seq(0, 320 , 30)) +
  scale_x_date(date_breaks = "3 months",
               date_labels = "%d %b %y") +
  guides(colour=guide_legend(override.aes = list(size = 2),
                             ncol = 3, keyheight = 0.5, keywidth = 0.5)) +
  theme(legend.position = "bottom",
        panel.background = element_rect(fill = "gray95"),
        axis.text.x = element_text(angle = 90,
                                   hjust = 1, 
                                   size = 7),
        axis.text.y = element_text(size = 7),
        legend.text = element_text(size = 7))
```

<img src="notebook_files/figure-gfm/fig2-1.png" width="100%" style="display: block; margin: auto;" />

### Time to go to bed

And at which hour in the day I read more often. This one is interesting
because I have the habit of reading before going to bed. But it also
happens that some insomnia may hit and I end up by picking up my Kindle
to take my mind out of whatever is preventing my sleep.

There are some clear issues regarding Daylight Saving Time, indicated by
sudden shifts in the hour in which I started reading, usually by October
and February. These are likely related to DST no longer being applied in
Brazil and this being ignored in the conversion I made. I???ve fought this
battle once, I don???t want to try to fix it again.

I also marked with vertical lines my time in the US. It looks like: I
went for my Kindle in the middle of the night a lot more in 2019 than in
the other years, I went to bed later while in the US and that Jane Eyre
was basically the only book that I read during the day to finish it
faster.

``` r
read_session_all <- read_session_ %>%
  select(start_timestamp, ASIN, total_reading_millis, title) %>%
  filter(title %in% unlist(topbooks)) %>%
  mutate(total_reading_millis = total_reading_millis / (1000 * 60),
         start_timestamp = gsub("Z", "", start_timestamp),
         start_timestamp = ymd_hms(start_timestamp),
         date_ = date(start_timestamp),
         start_timestamp_ = if_else((date_ >= as.Date("2019-08-12")) &
                                      (date_ <= as.Date("2020-04-09")),
                                       with_tz(start_timestamp, "US/Eastern"),
                                       with_tz(start_timestamp, "Etc/GMT-3")),
         hour_ = hour(start_timestamp_)) %>%
  left_join(appear_order) %>%
  arrange(order_) %>%
  mutate(title2 = factor(title,
                         levels = title,
                         labels = title,
                         ordered = TRUE))

ggplot(read_session_all) +
  geom_point(aes(x=date_, y=hour_,
                 col=title2),
             position="jitter") +
  geom_vline(aes(xintercept=as.Date("2019-08-12")), lty = 2) +
  geom_vline(aes(xintercept=as.Date("2020-04-09")), lty = 2) +
  labs(x = "Date", y = "Hour of the day", color="") +
  scale_discrete_manual(values = paletteer_d("ggsci::category20_d3"),
                        aesthetics = "colour") +
  scale_alpha_continuous(name="Duration",
                         breaks=c(0, 30),
                         labels=c(0, 30)) +
  scale_x_date(date_breaks = "3 months",
               date_labels = "%d %b %y") +
  guides(colour=guide_legend(override.aes = list(size = 2),
                             ncol = 3, keyheight = 0.5, keywidth = 0.5)) +
  theme(legend.position = "bottom",
        panel.background = element_rect(fill = "gray95"),
        axis.text.x = element_text(angle = 90,
                                   hjust = 1, 
                                   size = 7),
        axis.text.y = element_text(size = 7),
        legend.text = element_text(size = 7))
```

<img src="notebook_files/figure-gfm/fig3-1.png" width="100%" style="display: block; margin: auto;" />
