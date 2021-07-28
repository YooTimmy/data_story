---
title:  "Promo Code Scraper with Beautiful Soup and Flask"
date:   2021-06-07 12:39:59 +0800
categories:
  - Scraper
tags:
  - Web Crawling
  - Flask
  - Beautiful Soup
toc: true
toc_sticky: true
---

With more people working from home and adapting to the new norm, food deliveries and online shopping are becoming more and more popular. To increase sales,
many merchants are giving out vouchers or promo codes when customer purchases meeting certain criteria. However, it is a bit tedious to search for promo code
every time, and some websites are hiding the code with a picture, so that you have to click in to reveal the actual promo code. Moreover, there are
always a lot of terms and conditions associated with promo codes, such as promotion valid time, minimum spend and participating merchants, etc. All these information
are available there, however some lines are very easy to miss out due to formatting, etc.

Therefore, I came up with this idea to implement an handy tool to display the promo codes and their associated terms and conditions in a more readable format.

## Web Scraping

To begin with, we need to find out how to obtain the information we need. Instead of directly obtaining the data from official websites, there are also websites
updating the voucher promo codes collectively into 1 page with consistent formatting. For illustration purpose, we will demonstrate how to extract data from [this
site](https://www.sgdtips.com/grabfood-promo-codes), where Grab voucher code would be updated on a monthly level. Here Beautiful Soup library is used to help us
remove the unwanted HTML markups. With just a few lines of code, we could download the text information from the page:

```python
website = "https://www.sgdtips.com/grabfood-promo-codes"
res = requests.get(website)
soup = bs4.BeautifulSoup(res.text, "lxml")
```

Now the next step is really dive into the web page itself, locate the data element that we want and then find out how we are able to retrieve that piece of information
based on either HTML tag elements or CSS class. Just to give an example here as illustration purpose:

For instance, when extracting voucher codes information, we need to right click the voucher code in page, and select "Inspect" option to open up the page
where the HTML source code would show up. (Shown in the figure below) It is observed that we can obtain the voucher code value by tracing down the CSS class names
from "item.procoupon_item--voucher" -> "sgdtpro_voucher-content" -> "sgdt-brief-promo.promo-code-h3", which we can easily obtain with a nested for loop.

By now, you should also realize that the code for scraping web pages are not very likely to be reused, as for different website the tags, class names would vary for sure.
Nonetheless, similar code structure should still be expected, where we can loop through each relevant items and save the important information accordingly. Below is the code
block I used to extract relevant information for the target website.

```python
def promo_code_scraper(soup):
    """
    The web scraper code is very specific for each individual website, so likely it won't work if you change the
    website link to another one, or the website changes its format subsequently.
    :param soup: the output from beautifulsoup module
    :return: the voucher_df in pandas df format.
    """
    voucher_desc_lst = []
    voucher_code_lst = []
    voucher_detail_lst = []
    for item in soup.select(".item.procoupon_item--voucher"):
        for voucher in item.select(".sgdtpro_voucher-content"):
            for voucher_desc in voucher.select('.sgdt-brief-promo.promo-code-h3'):
                voucher_desc_lst.append(voucher_desc.text)
            if not voucher.select('.sgdt_code-value'):
                voucher_code_lst.append('NA')
            else:
                for code in voucher.select('.sgdt_code-value'):
                    # Here the get method helps to obtain the value filed in the hidden tag (.text is not working..)
                    voucher_code_lst.append(code.get('value'))
        for details in item.select(".sgdtpro_content-detail"):
            if not details.text.replace('\n', ''):
                voucher_detail_lst.append('NA')
            else:
                voucher_detail_lst.append(details.text.replace('\n', ''))

    voucher_df = pd.DataFrame(list(zip(voucher_desc_lst, voucher_code_lst, voucher_detail_lst)),
                              columns=['Description', 'Code', 'Details'])
    return voucher_df
```
## Display the data with Flask

Now we have extract the data we want. If it is only for our own consumptions, we could probably save it into excel and check it there. However, this time
I would like to try display the information in a web page, so that it could be potentially hosted online for more people to access.

Now what technology shall we use? Flask comes to my mind given its natural integration with python. Right now we have the data saved as a pandas DataFrame,
how could we show that in a web page with Flask? I have done some researches on this,
and [this blog](https://blog.miguelgrinberg.com/post/beautiful-interactive-tables-for-your-flask-templates) from Miguel seems to be a perfect match of what
I want to achieve here. In his posts, several different ways of rendering a table are explained. The most basic one is using the Boostrap table directly,
it is very simple to implement but it is only practical when the number of rows are small. Additionally, the interactive features for a table such as pagination,
searching and sorting are not there as well. That is where dataTables.js shines, where all these features could be configured with the help of it. I am not
an experts in CSS, so in case you are interested to know more in this i would recommend you follow through the tutorials and blog mentioned in the link earlier.

With the HTML page set-up in place, the final implemented page looks like this:

Thanks to dataTables.js, we can now rank the voucher code columns, search for key words in this page. This comes in handy if you are looking for the promo codes
for certain banks, say DBS or UOB. Now instead of having to making multiple clicks to review the hidden codes, you can have a direct view on the available
promotions in the latest month and what are the terms and conditions associated, etc.codes

## My Reflections

The idea of this project really comes along when I am searching for grab voucher code (again) when ordering food recently. After making several clicks and sometimes
closing up multiple ads associated to the buttons, I start to think: why not find a way to better organize this information? Prior to this project I have not really
applied the web scraping to a real projects, and in terms of flask and CSS they are all brand new things for me also. After going through this project, I have
picked up a lot of new knowledge, gained hands-on experience about techniques that I never applied before, etc. This type of "project oriented" learning has been quite
effective based on my experiences.

## Full Code

Lastly, you can find the full code base used for this project in my Github page [here](...).

