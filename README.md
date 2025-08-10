import streamlit as st
import pandas as pd
import keepa
import time
import csv
from io import StringIO

st.title("Amazon ASIN zu eBay CSV Converter")

st.markdown("""
Diese App holt Produktdaten von Amazon (via Keepa API) und erstellt eine eBay File Exchange CSV mit Gewinnaufschlag.
""")

uploaded_file = st.file_uploader("ASIN CSV-Datei hochladen (eine Spalte mit ASINs)", type=["csv", "txt"])
keepa_key = st.text_input("Keepa API Key")
markup_percent = st.number_input("Aufschlag in Prozent", min_value=0, max_value=200, value=30, step=1)
paypal_email = st.text_input("Deine PayPal Email für eBay Listings")
quantity = st.number_input("Menge", min_value=1, max_value=1000, value=1)
category_id = st.text_input("eBay Kategorie ID (optional)")
condition_id = st.number_input("Condition ID (1000=Neu)", min_value=1000, max_value=3000, value=1000)
shipping_cost = st.text_input("Versandkosten (z.B. 0.00)", value="0.00")

def read_asins(file):
    try:
        df = pd.read_csv(file, dtype=str)
        if 'ASIN' in df.columns:
            return df['ASIN'].dropna().astype(str).str.strip().tolist()
        else:
            # fallback: take first column
            return df.iloc[:,0].dropna().astype(str).str.strip().tolist()
    except Exception:
        # fallback if txt
        content = file.getvalue().decode("utf-8")
        lines = content.splitlines()
        return [l.strip() for l in lines if l.strip()]

def fetch_keepa_products(asins, key):
    api = keepa.Keepa(key)
    products = []
    for asin in asins:
        try:
            res = api.query(asin, domain='DE')
            p = res[0] if isinstance(res, list) else res
            title = p.get('title', '')[:80]
            price = None
            if p.get('buyBoxPrice'):
                price = p['buyBoxPrice'] / 100.0
            elif 'offers' in p and p['offers']:
                price = p['offers'][0].get('price', None)
                if price:
                    price = price / 100.0
            images = p.get('imagesCSV', '')
            img_url = ""
            if images:
                first = images.split(',')[0]
                img_url = f"https://m.media-amazon.com/images/I/{first}"
            products.append({
                'asin': asin,
                'title': title,
                'price': price,
                'image_url': img_url
            })
            time.sleep(1)
        except Exception as e:
            products.append({
                'asin': asin,
                'title': '',
                'price': None,
                'image_url': '',
                'error': str(e)
            })
    return products

def build_csv(products, markup, paypal, qty, catid, condid, shipcost):
    output = StringIO()
    writer = csv.DictWriter(output, fieldnames=['Title','Subtitle','Description','Start Price','Quantity','Category','ConditionID','PictureURL','PaymentMethods','PayPalEmailAddress','ShippingService-1:Cost'])
    writer.writeheader()
    for p in products:
        price = p['price']
        ebay_price = round(price * (1 + markup / 100), 2) if price else ""
        description = f"ASIN: {p['asin']}<br>Originalpreis: {price if price else 'n/a'} €"
        writer.writerow({
            'Title': p['title'] or p['asin'],
            'Subtitle': '',
            'Description': description,
            'Start Price': ebay_price,
            'Quantity': qty,
            'Category': catid,
            'ConditionID': condid,
            'PictureURL': p['image_url'],
            'PaymentMethods': 'PayPal',
            'PayPalEmailAddress': paypal,
            'ShippingService-1:Cost': shipcost
        })
    return output.getvalue()

if st.button("CSV generieren"):
    if not uploaded_file:
        st.error("Bitte ASIN CSV hochladen!")
    elif not keepa_key:
        st.error("Bitte Keepa API Key eingeben!")
    elif not paypal_email:
        st.error("Bitte PayPal Email eingeben!")
    else:
        asins = read_asins(uploaded_file)
        with st.spinner(f"Keepa-Daten abrufen für {len(asins)} ASINs..."):
            products = fetch_keepa_products(asins, keepa_key)
        csv_content = build_csv(products, markup_percent, paypal_email, quantity, category_id, condition_id, shipping_cost)
        st.download_button(label="CSV herunterladen", data=csv_content, file_name="ebay_fileexchange.csv", mime="text/csv")
