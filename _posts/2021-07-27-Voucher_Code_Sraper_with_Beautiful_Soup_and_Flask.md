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

To begin with, we need to find out how to obtain the data required. Instead of directly obtaining the data from official sites respectively, there are also websites
updating the voucher promo codes collectively with consistent formatting on a regular basis, which seems more suitable for our use case. For illustration purpose,
we will demonstrate how to extract data from [this site](https://www.sgdtips.com/grabfood-promo-codes), where Grab voucher code would be updated on a monthly basis.
Beautiful Soup library is used to help us remove the unwanted HTML markups. With just a few lines of code shown below, we could download the text information from the page.

```python
website = "https://www.sgdtips.com/grabfood-promo-codes"
res = requests.get(website)
soup = bs4.BeautifulSoup(res.text, "lxml")
```

Now the next step is really diving into the web page itself, locate the data elements we need and then find out how we are able to retrieve that piece of information
based on either HTML tags or CSS classes info. Just to give an example here for illustration purpose:

For instance, when extracting voucher codes data, we need to right click the voucher code in page, and select "Inspect" option to open up the page
where the HTML source code would show up. It is observed that we can obtain the voucher code value by tracing down the CSS class names with given sequences:
("item.procoupon_item--voucher" -> "sgdtpro_voucher-content" -> "sgdt-brief-promo.promo-code-h3"), which can be easily obtained with a nested for loop.

![](https://raw.githubusercontent.com/YooTimmy/data_story/gh-pages/assets/images/Voucher_Scraper/website_example.png)

By now, you should also realize that the code for scraping web pages is less likely to be directly reused, as for different website the tags/class names would vary for sure.
Nonetheless, similar code structure should still be expected, where we can loop through relevant items and save the important information accordingly. Below is the code
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
## Display the table with Flask

Now we have extracted the data we need from web pages. If it is only for our own consumptions, we could probably save it into excel and check it there. However, this time
I would like to try display the information in a web page, so that it could be potentially hosted online and accessible by more people.

Now what technologies shall we use? Flask comes to my mind first given its natural integration with python (though I have not really used it before).
Right now we have the data saved as a pandas DataFrame, how could we show that in a web page with Flask?

After doing some researches on this, [this blog](https://blog.miguelgrinberg.com/post/beautiful-interactive-tables-for-your-flask-templates) from Miguel seems to be a perfect match of what
we want to achieve here. In his posts, several different options of rendering tables in Flask are explained. The most basic one is using the Boostrap table directly.
It is very simple to implement, but it is only practical when the number of rows are small. Additionally, the interactive features for tables such as pagination,
searching and sorting are not there as well. That is where dataTables.js shines, where all these features could be configured with the help of this library. I am not
an experts in either Flask or CSS, so in case you are interested to know more about this, I would recommend you following through the tutorials and blog mentioned in the link earlier.

With the HTML page set-up in place, the final implemented page looks like this:

![](https://raw.githubusercontent.com/YooTimmy/data_story/gh-pages/assets/images/Voucher_Scraper/flask_table.png)

Thanks to dataTables.js, we can now do things like sorting the voucher code column, searching for key words in this page (This comes in handy if you are looking for the promo codes
for certain banks, say DBS or UOB), etc. What's more, now instead of having to make multiple clicks to review the hidden promo codes, you can have a direct view on the available
promotions in the latest month and what are the terms and conditions associated, etc.

## My Reflections

The idea of this project really came along when I was searching for grab voucher code (again) when ordering food recently. After making several clicks and
closing up multiple ads associated to the buttons, I start to think: why not finding a way to better organize this information? Prior to this project, I have not really
applied the web scraping to a actual project myself, and in terms of flask and CSS they are all new things for me as well. After going through this project, I have
picked up a lot of new knowledge, gained hands-on experience about the techniques that I never applied before, etc. This type of "project oriented" learning has been quite
effective based on my experiences.

## Full Code

Lastly, you can find the full code base used for this project in my Github page [here](https://github.com/YooTimmy/promo_code_scraper). Once getting the code, you can navigate
to the target folder and type "python promo_code_scraper.py", then you will have your promo code scraper running at http://127.0.0.1:5000/ locally.




