Introduction
------------

Ever wonder how automated movie recommender systems work: systems where
you rate a few movies, then other movies are recommended to you based on
your inferred preferences? Well, I've put together a documentation on
how I built one of my own in R without the use of black-box algorithms
or tools. I used the collaborative filtering technique. You can
modify/customize my workflow to suit your needs.

Getting Data
------------

To begin, you will need access to a database of movies, all of which
have been rated by other people. For simplicity, let's call these people
users. The users only have to rate some movies not all movies, but all
movies have to be rated by some users. Fortunately for us, these sorts
of databases exist. I will use the publicly available [MovieLens data
set collected by GroupLens
Research](!http://grouplens.org/datasets/movielens/). Because I want to
use a data set containing movies as recently released as posible, I will
use the [MovieLens Latest Dataset, ml-latest-small.zip (download date:
Jun 13, 2016)](!http://grouplens.org/datasets/movielens/latest/). The
ml-latest-small data set contains 105339 ratings across 10325 movies
provided by 668 users and was generated on January 11, 2016. The average
number of ratings per user is 158.

For files are included in the data set: `links.csv`, `movies.csv`,
`ratings.csv` and `tags.csv`. I am most interested in the `movies.csv`
and `ratings.csv` files.

User ratings are contained in the `ratings.csv` file which is a csv file
with columns representing

-   userId: user identification tag
-   movieId: movie identification tag
-   rating: user's rating (1-5 stars) of movie with specified movieId
-   timestamp: timestamp of when movie was rated

Movie titles are contained in the `movies.csv` file which is a csv file
with columns representing

-   movieId: movie identification tag
-   title: movie title
-   genres: movie genre

Import data to project:

    movies  <- read.csv("data/ml-latest-small/movies.csv",  header=T)
    ratings <- read.csv("data/ml-latest-small/ratings.csv", header=T)

Data Wrangling
--------------

After obtaining the movie ratings data set, the next step is to
transform it into the right format. For our purpose, the right format is
a matrix with rows representing individual movies and columns
representing individual users. The cells of this matrix contains the
rating of a movie represented by that row by the user represented by
that column. This transformation can be done in R using nested for
loops. Note that if done in R, the process will take a long time to
complete. I wrote a c++ code to accomplish the task in seconds. This c++
code, `main.cpp` is located in the `data_and_codes/` directory.

We also need to create a matrix that indicates which cells have ratings
in them. We will use the value 1 to represent cells with ratings and 0
otherwise.

R code:

    # get maximum user id number
    userId_max  <- max(ratings$userId)
    # get maximum movie id number
    movieId_max <- max(ratings$movieId)

    # create matrix with number of rows equal to maximum number
    # of movies and columns equal to maximum number of users 
    ratingsMatrix <- matrix(0,length(1:movieId_max),length(1:userId_max))
    ratedMatrix   <- matrix(0,length(1:movieId_max),length(1:userId_max))

    # loop through data and fill matrices
    for (i in 1:movieId_max) {
        for (j in 1:userId_max) {
            rown <- ratings[ratings$movieId == i & ratings$userId == j,]
            if (nrow(rown) > 0) {
                ratingsMatrix[i,j] <- rown$rating
                ratedMatrix[i,j]   <- 1
            }
        }
    }

    # get ids of rated movies
    movieIdColumn <- seq(1:nrow(ratingsMatrix))

If c++ code is used, read output files:

    ratingsMatrix <- read.csv("ml-latest-small_ratingsMatrix.csv", header=F)
    ratedMatrix   <- read.csv("ml-latest-small_ratedMatrix.csv",   header=F)

    # convert data frames to matrices
    ratingsMatrix <- as.matrix(ratingsMatrix)
    ratedMatrix   <- as.matrix(ratedMatrix)
    movieIdColumn <- seq(1:nrow(ratingsMatrix))

Because we created a matrix large enough to have row numbers as large as
the largest movieId, there are lots of empty rows or rows with only
zeros in our matrices. We should remove these rows.

    # remove empty rows
    ratingsMatrixFinal <- ratingsMatrix[as.logical(rowSums(ratingsMatrix != 0)), ]
    ratedMatrixFinal   <- ratedMatrix  [as.logical(rowSums(ratingsMatrix != 0)), ]
    movieIdColumnFinal <- movieIdColumn[as.logical(rowSums(ratingsMatrix != 0))  ]

Add My Ratings
--------------

After creating a matrix of ratings, I added my ratings of 53 movies. I
identified movies that I would rate 1, 2, 3, 4, and 5 stars. You can
find my ratings in the file `myratings_mod.csv` in the directory
`data_and_codes/`. This step can be done prior to data wrangling, but I
chose to do it after. Results will be the same.

    # import csv file containing my ratings
    myratingsraw <- read.csv("data_and_code/myratings_mod.csv",header=T)

    # find which rows represent the movie I rated
    findRow <- function(item,vec) which(vec==item)
    myratingsrows <- unlist(
        sapply(myratingsraw$movieId,
               findRow,movieIdColumnFinal))

    myratings <- numeric(nrow(Y))

    for (i in 1:length(myratingsrows)) {
        r <- myratingsrows[i]
        myratings[r] <- myratingsraw$rating[i]
    }

    # set ratingsMatrixFinal to Y and ratedMatrixFinal to R
    Y <- ratingsMatrixFinal
    R <- ratedMatrixFinal

    # append my ratings to Y and R
    myY <- cbind(myratings, Y)
    myR <- cbind(as.integer(myratings != 0), R)
        

    # compute the mean of ratings for each movie
    # we will subtract this value from all ratings 
    # before running collaborative filtering
    myY_sum <- rowSums(myY)
    myR_sum <- rowSums(R)
    Ymean   <- y_sum/r_sum

Collaborative Filtering Cost Function and Gradient
--------------------------------------------------

Since we will be using gradient descent in solving our collaborative
filtering algorithm, we need to define functions that compute our cost
function and gradient. This function takes as input, a regularization
parameter lambda, a matrix of movie rating indicators R, a matrix of
movie coefficients Theta, and a matrix of user coefficients X. We will
use collaborative filtering to compute the matrices X and Theta. Since
we don't know these at the onset, we will initialize them with random
numbers.

    collaborativeFilter <- function (X, Y, Theta, R, lambda) {
        # 1. cost function
        J <- 1/2 * ( (X%*%t(Theta)) - Y )^2 * R;
        
        reguTheta <- lambda/2 * sum(Theta^2) # Theta regularization
        reguX     <- lambda/2 * sum(X^2)     # X regularization
        
        J <-  sum(J) + reguTheta + reguX
        
        # 2. gradients
        X_grad     <- ((X%*%t(Theta)-Y)  * R) %*% Theta + lambda * X;
        Theta_grad <- t((X%*%t(Theta)-Y) * R) %*% X     + lambda * Theta;
        
        list(J,X_grad,Theta_grad)
    }

Execution
---------

We now have everything we need to run our algorithm. Let's put them
together.

### 1. Initialize matrices, vectors, and scalars

    # set Y to myY and R to myR
    Y <- myY
    R <- myR

    # mean normalize Y. i.e. subtract mean ratings
    for (i in 1:ncol(Y)) Y[,i] <- Y[,i] - Ymean

    # set scalars
    nmovies   <- nrow(Y) # number of movies
    nusers    <- ncol(Y) # number of users
    nfeatures <- 20

    lambda <- 1  # regularization parameter set to 1
    niter  <- 30000 # number of iterations
    alpha  <- 0.00004 # gradient descent step parameter

    costvec <- numeric(niter) # initialize vector of cost computation

    set.seed(1234) # set random number generator seed to 1234

    # initialize X to matrix of uniformly distributed random numbers
    X     <- matrix(runif(nmovies*nfeatures),nmovies,nfeatures)

    # initialize Theta to matrix of uniformly distributed random numbers
    Theta <- matrix(runif(nusers*nfeatures),nusers,nfeatures)

### 2. Run collaborative filtering with gradient descent

    for (i in 1:niter) {
        collabfilt <- collaborativeFilter(X,Y,Theta,R,lambda)
        
        costvec[i] <- collabfilt[[1]]
        X_grad     <- collabfilt[[2]]
        Theta_grad <- collabfilt[[3]]
        
        X     <- X     - alpha * X_grad
        Theta <- Theta - alpha * Theta_grad
    }

### 3. Extract movie recommendations

    # get movie titles in vector
    findMovieTitles <- function(item,vec) which(vec==item)
    mymovierows <- unlist(
        sapply(movieIdColumnFinal,findMovieTitles,movies$movieId))

    # compute my predictions and add column with movie titles
    p <- X %*% t(Theta) + Ymean

    # create data frame with predictions and movie titles
    mypred <- data.frame(movieTitle=movies$title[mymovierows],
                         movieId=movieIdColumnFinal,
                         rating=round(p[,1],1)
                         )

    # sort prediction by rating
    mypred_sort <- mypred[with(mypred, order(-rating)), ]

    # add movie release year to prediction data frame
    getMovieYear <- function(text) {
        y1  <- gsub("[^0-9]","",gsub("[^0-9\\(\\)]","",text))
        len <- nchar(y1)
        y2  <- substr(y1, len-3, len)
    }
    mypred_sort$movieYear <- unlist(sapply(mypred_sort$movieTitle,getMovieYear))
    mypred_sort$movieYear <- as.integer(mypred_sort$movieYear)

    # sort prediction by release year, then rating
    mypred_sort <- mypred_sort[with(mypred_sort, order(-movieYear,-rating)), ]

### 4. Display recommendations for top 10 movies released in 2013, 2014 and 2015

Show recommendations of movies that I have not rated but predicted to be
rated highly by me.

<table>
<thead>
<tr class="header">
<th align="left">Movie Title</th>
<th align="right">Predicted Rating</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">Air (2015)</td>
<td align="right">4.6</td>
</tr>
<tr class="even">
<td align="left">Scooby-Doo! And Kiss: Rock and Roll Mystery (2015)</td>
<td align="right">4.0</td>
</tr>
<tr class="odd">
<td align="left">Ant-Man (2015)</td>
<td align="right">3.9</td>
</tr>
<tr class="even">
<td align="left">The Jinx: The Life and Deaths of Robert Durst (2015)</td>
<td align="right">3.9</td>
</tr>
<tr class="odd">
<td align="left">Vacation (2015)</td>
<td align="right">3.9</td>
</tr>
<tr class="even">
<td align="left">Ashby (2015)</td>
<td align="right">3.9</td>
</tr>
<tr class="odd">
<td align="left">Louis C.K.: Live at The Comedy Store (2015)</td>
<td align="right">3.8</td>
</tr>
<tr class="even">
<td align="left">The End of the Tour (2015)</td>
<td align="right">3.8</td>
</tr>
<tr class="odd">
<td align="left">Mad Max: Fury Road (2015)</td>
<td align="right">3.7</td>
</tr>
<tr class="even">
<td align="left">Spring (2015)</td>
<td align="right">3.7</td>
</tr>
<tr class="odd">
<td align="left">Feast (2014)</td>
<td align="right">4.3</td>
</tr>
<tr class="even">
<td align="left">Guardians of the Galaxy (2014)</td>
<td align="right">4.2</td>
</tr>
<tr class="odd">
<td align="left">Before We Go (2014)</td>
<td align="right">4.2</td>
</tr>
<tr class="even">
<td align="left">Olive Kitteridge (2014)</td>
<td align="right">4.1</td>
</tr>
<tr class="odd">
<td align="left">Wild Tales (2014)</td>
<td align="right">4.1</td>
</tr>
<tr class="even">
<td align="left">That Awkward Moment (2014)</td>
<td align="right">4.0</td>
</tr>
<tr class="odd">
<td align="left">How to Train Your Dragon 2 (2014)</td>
<td align="right">4.0</td>
</tr>
<tr class="even">
<td align="left">Boxtrolls, The (2014)</td>
<td align="right">4.0</td>
</tr>
<tr class="odd">
<td align="left">Hector and the Search for Happiness (2014)</td>
<td align="right">4.0</td>
</tr>
<tr class="even">
<td align="left">Kid Cannabis (2014)</td>
<td align="right">3.9</td>
</tr>
<tr class="odd">
<td align="left">Palo Alto (2013)</td>
<td align="right">4.7</td>
</tr>
<tr class="even">
<td align="left">Syrup (2013)</td>
<td align="right">4.6</td>
</tr>
<tr class="odd">
<td align="left">Austenland (2013)</td>
<td align="right">4.6</td>
</tr>
<tr class="even">
<td align="left">Safe Haven (2013)</td>
<td align="right">4.5</td>
</tr>
<tr class="odd">
<td align="left">American Hustle (2013)</td>
<td align="right">4.5</td>
</tr>
<tr class="even">
<td align="left">Wolf of Wall Street, The (2013)</td>
<td align="right">4.4</td>
</tr>
<tr class="odd">
<td align="left">From One Second to the Next (2013)</td>
<td align="right">4.3</td>
</tr>
<tr class="even">
<td align="left">Dallas Buyers Club (2013)</td>
<td align="right">4.3</td>
</tr>
<tr class="odd">
<td align="left">50 Children: The Rescue Mission of Mr. And Mrs. Kraus (2013)</td>
<td align="right">4.1</td>
</tr>
<tr class="even">
<td align="left">Now You See Me (2013)</td>
<td align="right">4.1</td>
</tr>
</tbody>
</table>

These are definitely movies I would enjoy :)

Conclusions
-----------

To conclude, we have built a movie recommender system based on
collaborative filtering technique. This movie recommender system uses
provided movie ratings to make predictions on movie ratings of new
users. We added my ratings of 53 movies and used the recommender system
to make recommendations based on my computed movie preferences. A visual
assessment of the generated list shows to my that the recommendation
system we built works!
