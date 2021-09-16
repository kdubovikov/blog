---
layout: post
title:  "Blazing fast category encoding"
date:   2021-09-16 00:00:00
categories: datascience
---

Category encoding can make a difference between bad and good machine learning modes. In this post, we will learn about target encoding with the [blazing_encoders](https://github.com/kdubovikov/blazing-encoders/) library.

<!--more-->

# Tables, tables, tables

Tables are amongst the most popular data format for machine learning. If you work in the industry, you will likely find that most businesses struggle with monetizing their tabular data instead of working with unstructured data formats like images, audio, and text.

In the tabular world, classical machine learning approaches are king. Of course, you can use deep learning, but with a high probability, you will spend more time to get a worse model than you would with LightGBM or CatBoost, or even our 200-year-old friend linear regression.

# Variable types

When working with tables, you can meet two kinds of variables: continuous and categorical. Continuous variables can represent speed, height, weight, cost, and other numeric quantities. From a programmer's perspective, you would use `int`, `float`, or `double` types to represent them.

Categorical variables are like enumerations. They represent a discrete set of factors like sex, eye color, or currency. You can turn any continuous variable into categorical using binning. For example, we can turn age into age groups by assigning each group a fixed range. For example: 

* Group 1 is 0–7 years old
* Group 2 is 7–12 years old
* Group 3 is 12 and older

Converting a categorical variable into a continuous one is more tricky, so we will look into this later.

In real-world datasets, categorical variables are prevalent. The ease of creating them often leaves you with several categorical variables slicing each continuous quantity in every way imaginable. Also, regular business datasets are filled with categoricals. For instance, in warehousing, for each price and weight of an item, you get product category hierarchy, a set of dates, its location in a warehouse that can be naturally represented as a set of categorical variables.

# Category encoding

Sadly, machine learning algorithms understand only numbers. They do not know anything about categoricals. Take ordinary linear regression:

$$
f(x) = \mathbb{W}x + \mathbb{b}
$$

where $$\mathbb{W}$$ is a weight matrix, and $$\mathbb{b}$$ is a bias vector. If we were to estimate an item price given its weight, this would transform into

$$
\text{price} = \beta_0 x_{weight} + b_0
$$

where $$\beta_0$$ is a linear regression coefficient, and $$b_0$$ is a bias value.

But how can we introduce a categorical variable such as product category into this equation? To do this, we need to encode a categorical variable in a numeric format. The first idea that might come to mind is representing different categories as numbers: 1, 2, 3, and so on up to $N$. However, this is a wrong move since numbers introduce artificial ordering information and qualitative information about your categorical. 

$$
\text{price} = \beta_0 x_{weight} + \beta_1 x_{category} + b_0
$$

Could you try to find a good coefficient for $$\beta_1$$ when $$x_{category}$$ is encoded as a set of numbers $${1 … N}$$?

A better way is to perform a one-hot encoding: introduce $$N$$ new binary variables for each category. When a product belongs to category $$n$$, the variable $$x_n$$ will be set to 1, and others will be left at 0.

However, what if we have thousands of levels in our category. For example, our categorical variable represents a product ID of a large online store like Amazon. Then, one-hot encoding will blow up our feature matrix with thousands of new columns.

# Target encoding

Fortunately, there are alternative ways to deal with this problem. [category_encoders](https://contrib.scikit-learn.org/category_encoders/#) package contains lots of categorical variable encoding algorithms, and each of them has its tradeoffs. 

The one that is used more frequently in practice, and the simplest one, is called mean target encoding. The idea is to introduce information about our target variable into categorical levels. Consider that we have the following dataset:

| Product ID | # of items bought | 
|------------|-------------------|
| 195285     | 10                |
| 195285     | 10                |
| 158123     | 11                |
| 110581     | 20                |
| 110581     | 4                 |
| 110581     | 15                |

To perform mean target encoding we will group this dataset by product ID and replace each product ID with the corresponding mean of items bought for this specific product:

* ID 195285 = (10 + 10) / 2 = 10
* ID 158123 = 11
* ID 110581 = (20 + 4 + 15) / 3 = 13

We can replace each product id using this mapping:

| Product ID (encoded) | # of items bought | 
|----------------------|-------------------|
| 10                   | 10                |
| 10                   | 10                |
| 11                   | 11                |
| 13                   | 20                |
| 13                   | 4                 |
| 13                   | 15                |

Yet, there is one problem left. You may notice that we have only one observation for ID 158123. Thus, according to (the law of large numbers)[https://en.wikipedia.org/wiki/Law_of_large_numbers], some product IDs with lots of observations will have better mean estimates than those with fewer observations.

We can calculate a global mean of all items bought independent of product ID to correct this. Then, the fewer observations we have, the more we should shift our estimate to the global mean. This way, product IDs will small sales will be represented by global statistics, and the more they become popular, the more target encoding will shift to their local mean.

We can achieve this by smoothing our local mean with the global mean. The amount of smoothing will be dependent on how many observations do we have in a group. Let $$\mu_{local}$$ and $$mu_{global}$$ be local and global means accordingly, $$n_{id}$$ is the number of samples for product $$id$$, $$s$$ is a smoothing factor between 0 and 1.

To calculate the encoding $$te_{id}$$, we will use the following formula:

$$
te_{id} = \mu_{global} \times (1 - \text{smoothing}) + \mu_{local} \times \text{smoothing}
$$

In that way, we interpolate between $$\mu_{global}$$ and $$\mu_{local}$$. But how should we calculate this interpolation coefficient? It clearly should be dependent on the number of observations for each product.  

Let's calculate the amount of smoothing that will shift target encodincs of each category towards the global mean value over all rows. The less data each category has, the more it will tend towards global mean:

$$
1 - \frac{1}{1 + exp(\frac{1 - n_{id}}{s})}
$$


This approach gives our model relevant information about the target variable.


# The problem of speed
When you work with large datasets in Python's most popular data analysis tool – pandas, you will end up doing group by's with some additional logic to implement target encoding. The problem is that if you have many categorical columns and they have large cardinality. {% sidenote 1 "Cardinality is the term for the number of categories" %} For problems like this, pandas will scale poorly. It will consume lots of memory and make you wait while wasting CPU cycles.

The implementation of various category encoding algorithms in [category_encoders](https://contrib.scikit-learn.org/category_encoders/#) also uses pandas, so you will likely face scaling issues on large datasets.

# 🔥 Blazing Encoders 🔥
[blazing_encoders](https://github.com/kdubovikov/blazing-encoders/) is a Python library powered up by Rust implementation of the classic category encoding algorithm. The interface should be familiar for category_encoders users:

```Python
from blazing_encoders import TargetEncoder_f64
import numpy as np

if __name__ == '__main__':
    np.random.seed(42)
    data = np.random.randint(0, 10, (5, 5)).astype('float')
    target = np.random.rand(5)

    encoder = TargetEncoder_f64.fit(data, target, smoothing=1.0, min_samples_leaf=1)
    encoded_data = encoder.transform(data)
    print(encoded_data)
```

You can use two of the available classes: `TargetEncoder_f64`, and `TargetEncoder_f32` to control the balance between memory usage and numerical precision of your target encoding process.

Underneath, the library will share as much memory as possible so that overhead should be minimal. Also, it will parallelize target encoding computation so that the overall process will complete much faster.

## Perfornamce

[This simple benckmark](https://github.com/kdubovikov/blazing-encoders/blob/master/examples/benchmark_blazing_encoders.py) generates a random dataset with given number of rows, columns and unique categories in each column. Then, it measures the time it took to compute target encodings with `blazing_encoders` and `category_encoders`.

Tests show, that data-parallel implementation of `blazing_encoders` is significantly faster than `category_encoders`. 

For a dataset with 100 000 rows and 100 randomly generated columns with 100 categories in each `blazing_encoders` finish working about 9 times faster on my Macbook Pro.

If we will increase the number of categories and columns in the dataset to 1000 `blzaing_encoders` will finish data processing in abount 20 seconds. For `category_encoders` it took 23 minutes! As you can see, with large datasets and high-cardinality categorical columns using `blazing_encoders` will save lots of your time. 