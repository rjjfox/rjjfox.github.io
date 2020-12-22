---
layout: project
title: "What price should you charge for your Airbnb?"
languages: Python
skills: [python, data cleaning, feature engineering, data viz, machine learning]
---

Maybe you have a place that you're fortunate enough to be able to potentially put on Airbnb but you're not sure if it's worth the hassle. With Airbnb making host listing data publicly available, we can break down the nightly prices to understand what factors influence how much you can charge.

We'll go through that here, providing an easy guide to get a rough idea for how much to charge if you have a place you can rent out. The data is specific to Boston, however, much of the information in this post will give you a good idea for things to think about for any location.

![Boston loveliness ><](https://cdn10.bostonmagazine.com/wp-content/uploads/sites/2/2020/10/boston-foliage-social.jpg){: width='90%' style="padding: 10px 0"}

Note: Airbnb also gives pricing tips based on much more sophisticated models than I can accomplish here ([more details](https://www.vrmintel.com/inside-airbnbs-algorithm/)). Use this post though for a very rough guide to get you started.

Alternatively, the findings from this post apply to those looking to understand which filters to play around with when trying to find a cheap (or expensive?) place, other than using the price filter on the search page ¯*(ツ)*/¯.

<!--description-->

{% include toc.html %}

## The factors we'll be looking at

We'll use two datasets given by Airbnb:

- calendar.csv - listings with dates detailing whether they're available or not and how much they cost to rent on each date.
- listings.csv - lots of information about each listing including text descriptions, host details, number of bedrooms, bathrooms, location and more.

We merge the two datasets to provide one informative dataset that we can work with. Some of the features:

- Neighbourhood
- Property type, e.g. house, apartment, dorm, etc.
- Room type, i.e. entire place, private room or shared room.
- Time of the year
- Number of bedrooms, bathrooms, beds
- Information about the host such as number of listings, response time, location, superhost status
- Number of reviews and average rating
- Amenities
- Cancellation policy

Cleaning and manipulating the data into a format we can use takes up much of the time invovled in a project such as this but we'll move swiftly on to more exciting things.

### Dropping listings without reviews

Since we want to find a price for our new listing that travellers will think reasonable, we'll focus on the data for only the listings that are "successful". We'll remove listings without a review.

This has the added benefit of removing a few listings that some might define as outliers. Outliers can greatly impact a model's ability to make accurate predictions (linear regression is particularly sensitive to outliers) but shouldn't be removed just to make the data fit better. Data may look like outliers but unless if there has been an obvious mistake somehow, they are valid and shouldn't be excluded.

## What are the most important factors?

Without any fancy statistical models, some factors stand out as key in pricing your listing. The plots below provide a birds-eye view of some of the features. Pay attention to the scales used in these plots to understand the magnitude of the impacts.

_Note: click on the plots to to be able to get a closer look._

![Four-way plot]

Things we can spot:

- The type of listing has a clear impact with shared and private rooms never reaching the higher prices that you can charge when renting out your entire home/apt
- The location of your property is important but maybe not as important as some of the other features
- Properties with a strict cancellation policy tend to come with listings charging a higher price - but these listings also tend to have less reviews (not shown here)
- The number of bedrooms has a huge impact on the price you can charge - 5-bedroom properties have an average price of $600 a night vs less than $200 for studios or 1-bed properties
- The average nightly price varies by $20 depending on the time of the year with late summer/autumn being the most expensive months

![Three panel plot]

Not displayed here to try and keep the post length to a minimum, exploring some of the features, there's no evident relationship between the descriptions given by hosts about their listings and the price. The presence or lack of a description about the neighbourhood, transport options or about the space don't appear to impact the price that a listing is put up for. There's no evidence to suggest that listings with longer descriptions or more house rules charge higher prices or vice versa.

## How can you calculate a price?

To enable us to add tangible values to the differnt factors and create a formula for estimating a good price, we'll plug the data into a **linear regression** model. Linear Regression is one of the simplest statistical modelling techniques but also one of the most easily interpreted.

Fitting the data with a linear regression model, we are able to account for 60% of the variability in price. We might be able to account for more with a more sophisticated model but for a ballpark price to work with, this suits our needs just fine. The ballpark price comes with a fair amount of wiggle room, on average around $100. This means that we need to bear this in mind when considering what to charge.

Our prediction effort is certainly not going to be winning any competitions but a key strength to linear regression is that it returns coefficients for each of the features. These coefficients tell us the impact of each of the features in an immediately interpretable form. Below, I've pulled out the key features identified by the model.

### Calculating a price

The way to use the graphs below is to go through each of the different features and add/subtract the coefficients together to get a rough idea for what price to put your Airbnb up for. To this you then need to add the 'intercept' of the model, in our case $$-\$86$$. That means you need to subtract $$\$86$$ from the final price.

Similar to the trends we saw before, key features are the number of bedrooms, the neighbourhood, type of property and room and the time of the year. Your status as a local or an Airbnb superhost also can have some impact on the sort of prices travellers will pay.

![lin reg features]

#### An example

For example, say I have a two bed apartment in South End that I want to rent out in January. I would calculate

- $$+\$50$$ for the South End neighbourhood
- $$+\$20$$ because it's an apartment
- $$-\$20$$ since January is a cheap month
- $$+\$0$$ because I want to use a moderate cancellation policy
- $$+\$42$$ since I'll rent out the whole apartment
- $$+2\times\$63$$ for the two bedrooms
- $$+\$35$$ for the one bathroom
- $$+3\times\$6$$ for the three beds (there's a sofa bed)
- $$+6\times\$6$$ since the place accommodates 6 people
- $$-\$86$$ for the intercept

Therefore the model says I should put the place up for a price around $$\$221$$ a night. Again, bearing in mind the potential error margin here, I might think that my apartment is in a particularly nice part of South End and want to push the price up a bit. Or maybe it's a bit of a cheek calling it a two-bed and knock a few dollars off.

Feel free to go through this yourself to see how much you might put your place up for.

There are some clear drawbacks to this approach. For one thing, it's possible to plug in data to return a negative listing price implying that you should pay someone to come and stay in your dorm in West Roxbury in December. However, whilst Linear Regression in this case may not be the best method to accurately predict listing prices, it does provide an easily interpeted model that can be used (at least in some form) without even needing to use anything other than a pen and paper.

## Checking the most important factors

Linear regression has provided a sensible way for us to calculate a price. However, with only 60% of the variation accounted for using a linear model, let's see if we can improve on this using some more sophisticated models.

To pick out any potential non-linear relationships the data, we'll use a couple of tree-based ensemble methods. Tree-based methods give us the flexibility to fit non-linear relationships but they also tend to be very sensitive to small variations in the data. This is why we'll use ensemble methods to reduce this tendency to overfit. Tree-based methods also tell us clearly the key factors in picking a price.

![Tree based ><](https://images.unsplash.com/photo-1475359524104-d101d02a042b?ixid=MXwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHw%3D&ixlib=rb-1.2.1&auto=format&fit=crop&w=2314&q=80){: width='90%'}

Using **Random Forests** and **XGBoost**, two very popular competition winning machine learning methods, using the same factors as before, we can account for over 90% of the variation in the data and make much better predictions. However, their high complexity make it impossible to pass on the model without Python installed on your machine.

Random Forests is one of the most powerful Machine Learning algorithms, despite its simplicity. It is an ensemble of Decision Trees, taking the average prediction from multiple individual trees all trained on a different random subset of the training data.

<details>
<summary markdown="span">Click to see the code
</summary>

```python

>>> from sklearn.ensemble import RandomForestRegressor
>>>
>>> rf = RandomForestRegressor(n_estimators=20, n_jobs=-1)
>>> rf.fit(X_train, y_train)
>>> y_pred_train = rf.predict(X_train)
>>> y_pred_test = rf.predict(X_test)
>>>
>>> r2_train = r2_score(y_train, y_pred_train)
>>> r2_test = r2_score(y_test, y_pred_test)
>>> rmse_train = (mean_squared_error(y_train, y_pred_train))**0.5
>>> rmse_test = (mean_squared_error(y_test, y_pred_test))**0.5
>>>
>>> print(
>>>     'Train R-squared: {:.3f}\tTrain RMSE: ${:.2f}\
>>>     \nTest R-squared: {:.3f}\tTest RMSE: ${:.2f}'
>>>     .format(r2_train, rmse_train, r2_test, rmse_test)
>>> )

Train R-squared: 0.935   Train RMSE: $39.01
Test R-squared: 0.937    Test RMSE: $38.17

```

</details>
<br>

Fitting the data using Random Forests we can account for over 90% of the variance with no sign of overfitting! This is a 30% improvement over Linear Regression. We've greatly improved on our ability to predict the price for a listing but there's still a sizeable $40 residual error standard deviation. There's stil a room for error even with so much of the variability in our dataset accounted for.

We'll look at the key features driving the prediction in a moment. First we'll look at a second ensemble method.

### XGBoost

XGBoost stands for Extreme Gradient Boosting, a "boosting" ensemble method that works by sequentially adding predictors to an ensemble, each one correcting its predecessor. Another popular boosting method is AdaBoost, which stands for Adaptive Boosting.

<details>
<summary markdown="span">Click to see the code
</summary>

```python

>>> from xgboost import XGBRegressor
>>>
>>> xgb = XGBRegressor()
>>> xgb.fit(X_train, y_train)
>>> y_pred_train = xgb.predict(X_train)
>>> y_pred_test = xgb.predict(X_test)
>>>
>>> r2_train = r2_score(y_train, y_pred_train)
>>> r2_test = r2_score(y_test, y_pred_test)
>>> rmse_train = (mean_squared_error(y_train, y_pred_train))**0.5
>>> rmse_test = (mean_squared_error(y_test, y_pred_test))**0.5
>>>
>>> print(
>>>     'Train R-squared: {:.3f}\tTrain RMSE: ${:.2f}\
>>>     \nTest R-squared: {:.3f}\tTest RMSE: ${:.2f}'
>>>     .format(r2_train, rmse_train, r2_test, rmse_test)
>>> )

Train R-squared: 0.899    Train RMSE: $48.79
Test R-squared: 0.903     Test RMSE: $47.53

```

</details>
<br>

Again we are able to account for 90% of the variation in prices. Let's look at the most important features.

### Feature Importance

We've chosen tree-based models here not only for their flexibility but also their ability to assess feature importance. Feature importance is measured by how much the tree nodes use a particular feature to predict prices. Scikit-Learn scales the results so that the sum of all importances is equal to 1.

Let's see the top most important features according to the two ensemble methods.

![Ensemble features]

Both ensemble methods are aligned that the number of bedrooms, bathrooms and whether the listing is an entire home/apt or not are the most important features. These features account for 57% of the importance using either of the two ensemble methods, showing just how closely they agree. The rest of the features are each given small weights in adding the total up to 100%.

## Conclusion

We managed to account for 60% of the variance in Airbnb prices with our features using a straightforward linear regression model. Through this we can get a good idea for a ballpark figure to start from with a fair amount of wiggle room based on your particular place. The more sophisticated ensembled methods agreed the most important features but we still stick with linear regression to provide a simple method for getting a rough idea of what nightly price to put your place up for since there's no passing over ML models involved.

### Avoiding complexity

Through the models chosen and with the task at hand, we've managed to avoid some extra steps here in making our predictions. We did not need to scale our data since we used Linear Regression rather than one of the regularized methods and scaling did not impact the ensemble methods. Further, since we are only providing guidelines, we did not need to optimize our models with hyperparameter tuning. Our ensemble methods scored well enough by most standards without the added complexity.

### There will always be that _Je Ne Sais Quoi_

The models have given us some sensible guidelines for choosing how much to put a place up for on Airbnb but in each case, there was a sizeable error margin.

While our models do a great job at generalizing prices based on features, each listing is unique and what makes a traveller decide that your place is worth the money comes down to many features we won't be able to capture, although Airbnb do their best with their model.

Airbnb's suggested price when you start the process for renting your place out is much more sophisticated than ours, taking into account far more granular features and using clustering techniques. But it can't see the amazing feng shui you've got going on in your bedroom or the hanging plants in the hallway. The best method is most likely trial and error, using the findings above and Airbnb's suggested price as guidelines. Try various prices and see what works.

## See the code

If you'd like to see the code behind the data, it's available on [Github](https://github.com/rjjfox/airbnb-datasets) and also on [Kaggle](https://www.kaggle.com/ryanfox212/how-much-should-you-price-your-boston-airbnb).

[four-way plot]: https://www.kaggleusercontent.com/kf/49945777/eyJhbGciOiJkaXIiLCJlbmMiOiJBMTI4Q0JDLUhTMjU2In0..DOB42_frsHjkZn05BK0aFw._DBcxNZGpvEcXD7J9T8Fr9yYFgRHKstCdgwmBizonHw3rT7EjgXS0_PNDw3YZmoH67eAMVtv3muD02XWWdLGcVlVJ6iU9W10NbMOg_wDGoXNeqDSGX7E6_ku1sbTjT633xr5k8Ssi6ZgpmC59Uz6TIYfoJuLzm0nWqnblcH4H_7nsn20LiQieuAi9LIzEEyA2Ort2Gnat1tj8EvN3Xj8K1p8J7c3e8Nyo-IqoEb9OMr3aoBYnocwxS9XCvPS8wbZ4WV2k_S4Znxrq3fUcXp2tHvMpVKzgCpjKVt-wZPydbEd-c5FzXjygU_CVQPC-mwOrOPidk975R9eS5WUhr9hu3KSB7wVzH12qxR-SkuFmCcTbWInQ4T6_NBxF4t3PP2Q1CW_U3TnEDlPKrofbtkQkcfthRQc7kMsUQ0P5PlQTFWYSmMpMDRQKuYSUyUiucaT716MZvImR2uIPmpp9ind_q-5RbUQWH7lhFzJnDoqNmG-PdtDvGaoE0XWgCQG_pFRPyau0VqySoMuln6Qp97XEWtcp1shikTqeSAJkxjrwGhnyQIrEkWLPNgoNKy0EImsUV3TryA4TtbBtErBkhVGuB_TJK6woPBOMFJG_IzvBldCH9Nseux1qJ2_qbClaUUEKuoDfGwAiox4R9w1Ds3aDiF_Vp-BjCcmGDMt4C7sgaU.aW2JtXjeNVQzkRCa0THBvw/__results___files/__results___20_0.png
[three panel plot]: https://www.kaggleusercontent.com/kf/49945777/eyJhbGciOiJkaXIiLCJlbmMiOiJBMTI4Q0JDLUhTMjU2In0..DOB42_frsHjkZn05BK0aFw._DBcxNZGpvEcXD7J9T8Fr9yYFgRHKstCdgwmBizonHw3rT7EjgXS0_PNDw3YZmoH67eAMVtv3muD02XWWdLGcVlVJ6iU9W10NbMOg_wDGoXNeqDSGX7E6_ku1sbTjT633xr5k8Ssi6ZgpmC59Uz6TIYfoJuLzm0nWqnblcH4H_7nsn20LiQieuAi9LIzEEyA2Ort2Gnat1tj8EvN3Xj8K1p8J7c3e8Nyo-IqoEb9OMr3aoBYnocwxS9XCvPS8wbZ4WV2k_S4Znxrq3fUcXp2tHvMpVKzgCpjKVt-wZPydbEd-c5FzXjygU_CVQPC-mwOrOPidk975R9eS5WUhr9hu3KSB7wVzH12qxR-SkuFmCcTbWInQ4T6_NBxF4t3PP2Q1CW_U3TnEDlPKrofbtkQkcfthRQc7kMsUQ0P5PlQTFWYSmMpMDRQKuYSUyUiucaT716MZvImR2uIPmpp9ind_q-5RbUQWH7lhFzJnDoqNmG-PdtDvGaoE0XWgCQG_pFRPyau0VqySoMuln6Qp97XEWtcp1shikTqeSAJkxjrwGhnyQIrEkWLPNgoNKy0EImsUV3TryA4TtbBtErBkhVGuB_TJK6woPBOMFJG_IzvBldCH9Nseux1qJ2_qbClaUUEKuoDfGwAiox4R9w1Ds3aDiF_Vp-BjCcmGDMt4C7sgaU.aW2JtXjeNVQzkRCa0THBvw/__results___files/__results___22_0.png
[lin reg features]: https://www.kaggleusercontent.com/kf/49945777/eyJhbGciOiJkaXIiLCJlbmMiOiJBMTI4Q0JDLUhTMjU2In0..DOB42_frsHjkZn05BK0aFw._DBcxNZGpvEcXD7J9T8Fr9yYFgRHKstCdgwmBizonHw3rT7EjgXS0_PNDw3YZmoH67eAMVtv3muD02XWWdLGcVlVJ6iU9W10NbMOg_wDGoXNeqDSGX7E6_ku1sbTjT633xr5k8Ssi6ZgpmC59Uz6TIYfoJuLzm0nWqnblcH4H_7nsn20LiQieuAi9LIzEEyA2Ort2Gnat1tj8EvN3Xj8K1p8J7c3e8Nyo-IqoEb9OMr3aoBYnocwxS9XCvPS8wbZ4WV2k_S4Znxrq3fUcXp2tHvMpVKzgCpjKVt-wZPydbEd-c5FzXjygU_CVQPC-mwOrOPidk975R9eS5WUhr9hu3KSB7wVzH12qxR-SkuFmCcTbWInQ4T6_NBxF4t3PP2Q1CW_U3TnEDlPKrofbtkQkcfthRQc7kMsUQ0P5PlQTFWYSmMpMDRQKuYSUyUiucaT716MZvImR2uIPmpp9ind_q-5RbUQWH7lhFzJnDoqNmG-PdtDvGaoE0XWgCQG_pFRPyau0VqySoMuln6Qp97XEWtcp1shikTqeSAJkxjrwGhnyQIrEkWLPNgoNKy0EImsUV3TryA4TtbBtErBkhVGuB_TJK6woPBOMFJG_IzvBldCH9Nseux1qJ2_qbClaUUEKuoDfGwAiox4R9w1Ds3aDiF_Vp-BjCcmGDMt4C7sgaU.aW2JtXjeNVQzkRCa0THBvw/__results___files/__results___39_0.png
[ensemble features]: https://www.kaggleusercontent.com/kf/49945777/eyJhbGciOiJkaXIiLCJlbmMiOiJBMTI4Q0JDLUhTMjU2In0..DOB42_frsHjkZn05BK0aFw._DBcxNZGpvEcXD7J9T8Fr9yYFgRHKstCdgwmBizonHw3rT7EjgXS0_PNDw3YZmoH67eAMVtv3muD02XWWdLGcVlVJ6iU9W10NbMOg_wDGoXNeqDSGX7E6_ku1sbTjT633xr5k8Ssi6ZgpmC59Uz6TIYfoJuLzm0nWqnblcH4H_7nsn20LiQieuAi9LIzEEyA2Ort2Gnat1tj8EvN3Xj8K1p8J7c3e8Nyo-IqoEb9OMr3aoBYnocwxS9XCvPS8wbZ4WV2k_S4Znxrq3fUcXp2tHvMpVKzgCpjKVt-wZPydbEd-c5FzXjygU_CVQPC-mwOrOPidk975R9eS5WUhr9hu3KSB7wVzH12qxR-SkuFmCcTbWInQ4T6_NBxF4t3PP2Q1CW_U3TnEDlPKrofbtkQkcfthRQc7kMsUQ0P5PlQTFWYSmMpMDRQKuYSUyUiucaT716MZvImR2uIPmpp9ind_q-5RbUQWH7lhFzJnDoqNmG-PdtDvGaoE0XWgCQG_pFRPyau0VqySoMuln6Qp97XEWtcp1shikTqeSAJkxjrwGhnyQIrEkWLPNgoNKy0EImsUV3TryA4TtbBtErBkhVGuB_TJK6woPBOMFJG_IzvBldCH9Nseux1qJ2_qbClaUUEKuoDfGwAiox4R9w1Ds3aDiF_Vp-BjCcmGDMt4C7sgaU.aW2JtXjeNVQzkRCa0THBvw/__results___files/__results___48_0.png
