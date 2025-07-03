# Zomato-Data-Analysis-and-Visualization using Python, Pandas and Plotly.

import pandas as pd
import matplotlib.pyplot as plt
import plotly.express as px
import seaborn as sns
import streamlit as st

# ---------- Data Loading and Preprocessing ----------
@st.cache_data
def load_data():
    try:
        df = pd.read_csv(r"C:\Users\theka\OneDrive\Desktop\MyProjects\zomato.csv")
        df1 = pd.read_excel(r"C:\Users\theka\OneDrive\Desktop\MyProjects\Country-Code.xlsx")
        return pd.merge(df, df1, on='Country Code', how='left')
    except FileNotFoundError:
        st.error("Make sure 'zomato.csv' and 'Country-Code.xlsx' are in the same directory.")
        return None

def convert_to_rupees(df2):
    exchange_rates = {
        'India': 1.0, 'United States': 74.50, 'United Kingdom': 101.20, 'South Africa': 5.00,
        'UAE': 20.30, 'Brazil': 13.40, 'New Zealand': 51.00, 'Australia': 55.00, 'Canada': 59.00,
        'Qatar': 20.40, 'Singapore': 55.30, 'Turkey': 8.30, 'Indonesia': 0.0052,
        'Philippines': 1.50, 'Sri Lanka': 0.37, 'Czech Republic': 3.50
    }
    df2['Amount in Rupees'] = df2.apply(
        lambda row: row['Average Cost for two'] * exchange_rates.get(row['Country'], 1.0), axis=1
    )
    return df2

# ---------- Visualization Functions ----------
def plot_avg_cost_by_country(df2):
    grouped = df2.groupby("Country")['Amount in Rupees'].mean().reset_index()
    fig, ax = plt.subplots(figsize=(12, 6))
    sns.barplot(x="Country", y='Amount in Rupees', data=grouped, ax=ax)
    ax.set_xlabel("Country")
    ax.set_ylabel('Average Cost for Two (in Rupees)')
    ax.set_title('Comparison of Average Cost for Two Across Countries')
    plt.xticks(rotation=45, ha='right')
    st.pyplot(fig)

def show_country_details(df2):
    selected_country = st.selectbox('Select a Country', df2['Country'].unique())
    country_data = df2[df2['Country'] == selected_country]

    # Cuisine vs Cost Bar Chart
    st.header(f'Cuisine vs. Amount in Rupees in {selected_country}')
    cuisine_cost = country_data.groupby('Cuisines')['Amount in Rupees'].mean().reset_index()
    fig = px.bar(cuisine_cost, x='Cuisines', y='Amount in Rupees',
                 title=f'Average Amount in Rupees per Cuisine in {selected_country}')
    st.plotly_chart(fig)

    # Online delivery vs Dine-in Pie Chart
    st.header(f'Online Delivery vs. Dine-in in {selected_country}')
    valid_country_data = country_data[country_data['Has Online delivery'].isin(['Yes', 'No'])].copy()
    valid_country_data['Delivery Type'] = valid_country_data['Has Online delivery'].map({'Yes': 'Online Delivery', 'No': 'Dine-in'})
    pie_data = valid_country_data['Delivery Type'].value_counts().reset_index()
    pie_data.columns = ['Delivery Type', 'Count']
    st.write(pie_data)
    if not pie_data.empty:
        fig_pie = px.pie(pie_data,values='Count',names='Delivery Type',title=f'Online Delivery Status in {selected_country}',
                         color='Delivery Type',color_discrete_map={'Online Delivery': 'orange','Dine-in': 'lightblue'})
        st.plotly_chart(fig_pie)
    else:
        st.info(f"No online delivery or dine-in data available for {selected_country}.")

def show_costly_cuisines_in_india(df2):
    if st.button('Show Costly Cuisines in India'):
        india_data = df2[df2['Country'] == 'India']
        cuisine_cost = india_data.groupby('Cuisines')['Amount in Rupees'].mean().reset_index()
        top_cuisines = cuisine_cost.sort_values('Amount in Rupees', ascending=False).head(5)

        st.subheader('Top 5 Costliest Cuisines in India')
        fig = px.bar(top_cuisines, x='Cuisines', y='Amount in Rupees',
                     title='Costliest Cuisines in India')
        st.plotly_chart(fig)

def analyze_indian_cities(df2):
    st.header('Analysis for Cities in India')
    india_data = df2[df2['Country'] == 'India']
    cities = india_data['City'].dropna().unique()
    selected_city = st.selectbox('Select a City in India', cities)

    city_data = india_data[india_data['City'] == selected_city]
    if not city_data.empty:
        st.subheader(f'Insights for {selected_city}')

        # Famous Cuisines
        st.markdown('**Famous Cuisines:**')
        popular = city_data['Cuisines'].str.split(', ').explode().value_counts().head(5)
        st.write(', '.join(popular.index))

        # Costlier Cuisine
        st.markdown('**Costlier Cuisine:**')
        costliest = city_data.groupby('Cuisines')['Amount in Rupees'].mean().reset_index()
        top = costliest.sort_values('Amount in Rupees', ascending=False).iloc[0]
        st.write(f"{top['Cuisines']} ({top['Amount in Rupees']:.2f} INR)")

        # Rating Count
        st.markdown('**Rating Counts:**')
        ratings = city_data['Rating text'].value_counts().reset_index()
        ratings.columns = ['Rating Text', 'Count']
        st.dataframe(ratings)

        # Pie Chart: Online vs Dine-in
        st.markdown('**Online Delivery vs. Dine-in:**')
        valid_delivery_data = city_data[city_data['Has Online delivery'].isin(['Yes', 'No'])].copy()
        valid_delivery_data['Delivery Type'] = valid_delivery_data['Has Online delivery'].map({'Yes': 'Online Delivery', 'No': 'Dine-in'})
        pie_data = valid_delivery_data['Delivery Type'].value_counts().reset_index()
        pie_data.columns = ['Delivery Type', 'Count']
        st.write(pie_data)
        if not pie_data.empty:
            fig = px.pie(pie_data, values='Count', names='Delivery Type',
                 title=f'Online Delivery vs. Dine-in in {selected_city}',
                 color_discrete_sequence=px.colors.qualitative.Set2)
            st.plotly_chart(fig)
        else:
            st.info("No online delivery or dine-in data available for this city.")

def india_report(df2):
    st.header('India-wide Report')
    india_data = df2[df2['Country'] == 'India']

    # More on Online Delivery
    st.subheader('Top Cities by Online Delivery Spending')
    online = india_data[india_data['Has Online delivery'] == "Yes"].groupby('City')['Amount in Rupees'].sum().reset_index()
    st.dataframe(online.sort_values('Amount in Rupees', ascending=False).head())

    # More on Dine-in
    st.subheader('Top Cities by Dine-in Spending')
    dine = india_data[india_data['Has Online delivery'] == "No"].groupby('City')['Amount in Rupees'].sum().reset_index()
    st.dataframe(dine.sort_values('Amount in Rupees', ascending=False).head())

    # High vs Low Living Cost
    st.subheader('Top 3 High and Low Cost Cities')
    avg_cost = india_data.groupby('City')['Amount in Rupees'].mean().reset_index()
    st.markdown('**High Living Cost Cities:**')
    st.dataframe(avg_cost.sort_values('Amount in Rupees', ascending=False).head(3))
    st.markdown('**Low Living Cost Cities:**')
    st.dataframe(avg_cost.sort_values('Amount in Rupees').head(3))

# ---------- Main ----------
def main():
    st.title(":rainbow[Zomato Data Analysis and Visualization]") 
    st.header('Comparison of Average Cost for Two Across Countries (in Rupees)')

    df2 = load_data()
    if df2 is not None:
        df2 = convert_to_rupees(df2)
        plot_avg_cost_by_country(df2)
        show_country_details(df2)
        show_costly_cuisines_in_india(df2)
        analyze_indian_cities(df2)
        india_report(df2)

if __name__ == "__main__":
    main()

