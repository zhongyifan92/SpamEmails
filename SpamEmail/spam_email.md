# Using Statistics to Identify Spam
Yi Zheng, Yifan Zhong, Yan Xu  
December 6, 2015  


###Introduction
Spam filters are used to examine various characteristics of an email before deciding whether to place it in the inbox or spam folder. In this project, we are going to develop and test spam filters by examining messages which have been classified by SpamAssassin. First, we summarize all the words occurring in a message and compare the frequencies of these words in ham and spam and then derive variables from characteristics of the message and use these to classify email. Next, we bring the email into R for processing. For each message, we locate its header, the messages itself and the attachments. Then we use the naive Bayes method to approximate the likelihood a message is spam given the message content and prepare the email for analysis. We will also use a decision tree to classify the message and carry out the analysis. Finally, we apply a recursive partitioning method to build a decision tree from the derived set of features.

###Reading in email messages
This part is all about reading the email Messages into R. We first find the directory in which the messages are stored. Then we list all the subdirectories. Then we use length() function to find out how many files are there all together in different subdirectories and how many in different subdirectories separately. Then we use readLines() function to read in the first file to check whether the message is read into R properly or not. We will take a sample of messages for testing the functions.

```r
#spamPath = "C:/Users/Yi/Desktop/SpamAssassinTrain"
spamPath = paste(getwd(),"/SpamAssassinTrain",sep="")
#list subdirectories
list.dirs(spamPath, full.names = FALSE)
```

```
## [1] ""           "easy_ham"   "easy_ham_2" "hard_ham"   "spam"      
## [6] "spam_2"
```

```r
#check total length 
dirNames = list.files(path = paste(spamPath, sep = .Platform$file.sep))
length(list.files(paste(spamPath, dirNames, sep = .Platform$file.sep)))
```

```
## [1] 9353
```

```r
#length of each subdirectories
sapply(paste(spamPath, dirNames, sep = .Platform$file.sep), function(dir) length(list.files(dir)) )
```

```
##   /Users/Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/easy_ham 
##                                                                         5052 
## /Users/Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/easy_ham_2 
##                                                                         1401 
##   /Users/Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/hard_ham 
##                                                                          501 
##       /Users/Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/spam 
##                                                                         1001 
##     /Users/Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/spam_2 
##                                                                         1398
```

```r
#list each of message from full directories
fullDirNames = paste(spamPath, dirNames, sep = .Platform$file.sep)
#read first message
fileNames = list.files(fullDirNames[1], full.names = TRUE)
fileNames[1]
```

```
## [1] "/Users/Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/easy_ham/00001.7c53336b37003a9286aba55d2945844c"
```

```r
#take a sample and read in lines
indx = c(1:5, 15, 27, 68, 69, 329, 404, 427, 516, 852, 971)
fn = list.files(fullDirNames[1], full.names = TRUE)[indx]
sampleEmail = sapply(fn, readLines)
```

###Naive Bayes Classification
The first approach we are going to use is Naive Bayes, which is a probability-based approach to classification. We'll perform Naive Bayes on all the collection of email messages that have already been read and classified as spam or ham. The Bayes Rule is:
P(message is spam | message content) = P(message content | spam)P(spam)/P(message content).
We assume that the chance a particular word is in the message is independent of all other words in the message, which greatly simplifies the computations.
Suppose the set of unique words ranges from "apple" to "zebra" as viewed. Then naive assumption says the likelihood of message content is given by:
P(message content | spam) = P(not apple | spam)...P(high | spam)...P(taxes | spam)...P(not zebra | spam)
It is the product of the probability of the word is absent or present given the message is spam. We can apply the similar function on ham mails.
Now the classification of a message depends on the likelihood ratio:
P(message content | spam)/ P(message content | ham)
We then take the log of the likelihood ratio:
log(P(message content | spam)/ P(message content | ham))
To avoid the case that the probability of a given word appears in spam is 0, we will smooth all the counts by adding 0.5 to them,
i.e. : P(high | spam) ~ (count of spam with high +1/2)/ (count of spam messages +1/2).
Therefore, for a new message we compute the log likelihood ratio as:
sum of logP(word present | spam) - logP(word present | ham) + sum of logP(word abesent | spam) - logP(word abesent | ham) + logP(spam) - logP(ham)
We will then find the threshold which if the log likelihood exceeds the threshold, it will be classified as spam.
In the following steps, we will:
Transform the message body into a set of words
Combine words across messages into a bag of words
Estimate the probability of a word appears in a message given it is spam or ham
Estimate the likelihood that a new text message is spam or ham given its content
Find a threshold for the log likelihood ratio.

###Splitting the Message into Its Header and Body
We write a function to split the message into its header and body. Noticing that the header and body are separated by an empty line. We only need to find the first empty line in the email as the split point.

```r
#split message into head and body
splitMessage = function(msg) {
  splitPoint = match("", msg)
  header = msg[1:(splitPoint-1)]
  body = msg[ -(1:splitPoint) ]
  return(list(header = header, body = body))
}
sampleSplit = lapply(sampleEmail, splitMessage)
```

###Removing Attachments from the Message Body
We first find the Content-Type key and use its value to determine whether or not an attachment is present. If so, we will find the boundary string and use this string to locate the attachments. For the missing Content-Type key, we will return FALSE in such case.

```r
headerList = lapply(sampleSplit, function(msg) msg$header)
hasAttach = sapply(headerList, function(header) {
  CTloc = grep("Content-Type", header)
  if (length(CTloc) == 0) return(FALSE)
  grepl("multi", tolower(header[CTloc])) 
})
```

We write this function to extract the boundary string from the messages that have attachments in order to locate and remove the attachments. We need to pay attention to the quotes and semicolon and we can use gsub() to get rid of them.

```r
#extract boundary string
getBoundary = function(header) {
  boundaryIdx = grep("boundary=", header)
  boundary = gsub('"', "", header[boundaryIdx])
  gsub(".*boundary= *([^;]*);?.*", "\\1", boundary)
}
```

We find that each body part has a structure that mimics the structure of the message with header information separated from the content with a blank line. We also find that some messages do not have attachment even though their headers indicates what they have. In order to drop the attachment, we will:
Drop the blank lines before the first boundary string
Keep the lines following the closing string as part of the first portion of the body and not the attachments
Use the last line of the email as the end of the attachment if we do not find closing boundary string.

```r
#drop the attachment only keep the body
dropAttach = function(body, boundary){
  #start of body/ start of a single attachment
  bString = paste("--", boundary, sep = "")
  bStringLocs = which(bString == body)
  #the case have no attachment 
  if (length(bStringLocs) <= 1) return(body)
  #closing boundary
  eString = paste("--", boundary, "--", sep = "")
  eStringLoc = which(eString == body)
  #the case that have attachment
  if (length(eStringLoc) == 0) 
    return(body[ (bStringLocs[1] + 1) : (bStringLocs[2] - 1)])
  
  n = length(body)
  #the case that have attachment in the middle, locate the part before and after the attachment
  if (eStringLoc < n) 
    return( body[ c( (bStringLocs[1] + 1) : (bStringLocs[2] - 1), 
                     ( (eStringLoc + 1) : n )) ] )
  
  return( body[ (bStringLocs[1] + 1) : (bStringLocs[2] - 1) ])
}
```

###Extracting Words from the Message Body
We first need to process the text, we begin with substituting all punctuation and digits with a blank and put all the words into lowercase. And we do the same thing on stopwords, which are not needed to be analyzed.

```r
#substitute all punctuation and digits with blank and uncapitalize the words
cleanText =
  function(msg)   {
    tolower(gsub("[[:punct:]0-9[:space:][:blank:]]+", " ", msg))
  }
#clean stopwords
library(tm)
```

```
## Loading required package: NLP
```

```r
stopWords = stopwords()
cleanSW = tolower(gsub("[[:punct:]0-9[:blank:]]+", " ", stopWords))
SWords = unlist(strsplit(cleanSW, "[[:blank:]]+"))
SWords = SWords[ nchar(SWords) > 1 ]
stopWords = unique(SWords)
```

We then remove the stop words in the message by dropping the collection of words appeared in stopwords from the message. We also drop one letter word. The return value from this function is a list of unique words in the messages.

```r
findMsgWords = 
  function(msg, stopWords) {
    #blank body
    if(is.null(msg))
      return(character())
    
    words = unique(unlist(strsplit(cleanText(msg), "[[:blank:]\t]+")))
    # drop empty and 1 letter words
    words = words[ nchar(words) > 1]
    words = words[ !( words %in% stopWords) ]
    invisible(words)
  }
```

We can combine the previous functions into a function called processAllWords(). 

```r
processAllWords = function(dirName, stopWords)
{
  # read all files in the directory
  fileNames = list.files(dirName, full.names = TRUE)
  # drop files that are not email, i.e., cmds
  notEmail = grep("cmds$", fileNames)
  if ( length(notEmail) > 0) fileNames = fileNames[ - notEmail ]
  #read in messages
  messages = lapply(fileNames, readLines, encoding = "latin1")
  
  # split header and body
  emailSplit = lapply(messages, splitMessage)
  # put body and header in own lists
  bodyList = lapply(emailSplit, function(msg) msg$body)
  headerList = lapply(emailSplit, function(msg) msg$header)
  rm(emailSplit)
  
  # determine which messages have attachments
  hasAttach = sapply(headerList, function(header) {
    CTloc = grep("Content-Type", header)
    if (length(CTloc) == 0) return(0)
    #check if the Content-Type is multipart
    multi = grep("multi", tolower(header[CTloc])) 
    if (length(multi) == 0) return(0)
    multi
  })
  
  hasAttach = which(hasAttach > 0)
  
  # find boundary strings for messages with attachments
  boundaries = sapply(headerList[hasAttach], getBoundary)
  
  # drop attachments from message body
  bodyList[hasAttach] = mapply(dropAttach, bodyList[hasAttach], 
                               boundaries, SIMPLIFY = FALSE)
  
  # extract words from body
  msgWordsList = lapply(bodyList, findMsgWords, stopWords)
  
  invisible(msgWordsList)
}
```

Before we select test and training data, we apply processAllWords() on all messages and get a list of number of unique words in all messages. We also get the total number of spam and ham messages.

```r
msgWordsList = lapply(fullDirNames, processAllWords, 
                      stopWords = stopWords) 
```

```
## Warning in FUN(X[[i]], ...): incomplete final line found on '/Users/
## Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/hard_ham/
## 00228.0eaef7857bbbf3ebf5edbbdae2b30493'
```

```
## Warning in FUN(X[[i]], ...): incomplete final line found on '/Users/
## Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/hard_ham/
## 0231.7c6cc716ce3f3bfad7130dd3c8d7b072'
```

```
## Warning in FUN(X[[i]], ...): incomplete final line found on '/Users/
## Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/hard_ham/
## 0250.7c6cc716ce3f3bfad7130dd3c8d7b072'
```

```r
numMsgs = sapply(msgWordsList, length)

#get the number and whether or not is spam for each message
isSpam = rep(c(FALSE, FALSE, FALSE, TRUE, TRUE), numMsgs)
#get a list of words from all messages
msgWordsList = unlist(msgWordsList, recursive = FALSE)

numEmail = length(isSpam)
numSpam = sum(isSpam)
numHam = numEmail - numSpam
```

###Test and Training Data
To make it reproducible, we first set seed. We choose to use 2/3 of the data for training and 1/3 for testing. We then determine the indices of test spam and ham message and then use the indices to select the word vectors from msgWordsList(). We finally use rep() to organize the test and training collections.

```r
set.seed(418910)
#set testdata
testSpamIdx = sample(numSpam, size = floor(numSpam/3))
testHamIdx = sample(numHam, size = floor(numHam/3))
#get two lists of all the words from testdata and traning data. Organized with spam messages first and ham next  
testMsgWords = c((msgWordsList[isSpam])[testSpamIdx],
                 (msgWordsList[!isSpam])[testHamIdx] )
trainMsgWords = c((msgWordsList[isSpam])[ - testSpamIdx], 
                  (msgWordsList[!isSpam])[ - testHamIdx])
#define whether is spam
testIsSpam = rep(c(TRUE, FALSE), 
                 c(length(testSpamIdx), length(testHamIdx)))
trainIsSpam = rep(c(TRUE, FALSE), 
                  c(numSpam - length(testSpamIdx), 
                    numHam - length(testHamIdx)))
```

We will find a complete listing of all words first and we find there are more than 80,000 unique words in our training subset.

```r
#summary unique words
bow = unique(unlist(trainMsgWords))
length(bow)
```

```
## [1] 80481
```

###Calculating Log Likelihood Ratio
For each word in bow, we compute the number of spam messages in our training set that contain the word. We process each message and retrieve only the unique words. This avoids any repeated words from any one mail. Then we collapse these words from across all the spam messages and calculate the frequency table. Since the values are all stored in matrix, we will select values corresponding to the words that appear in a new message and the words that are absent from the message. And we can use these to compute the log likelihood ratio for the message. Therefore, the trainTable can be used to construct the log likelihood ratio for a new message.

```r
#prob estimates from training data
computeFreqs =
  function(wordsList, spam, bow = unique(unlist(wordsList)))
  {
    # create a matrix for spam, ham, and log odds
    wordTable = matrix(0.5, nrow = 4, ncol = length(bow), 
                       dimnames = list(c("spam", "ham", 
                                         "presentLogOdds", 
                                         "absentLogOdds"),  bow))
    
    # For each spam message, add 1 to counts for words in message
    counts.spam = table(unlist(lapply(wordsList[spam], unique)))
    wordTable["spam", names(counts.spam)] = counts.spam + .5
    
    # Similarly for ham messages
    counts.ham = table(unlist(lapply(wordsList[!spam], unique)))  
    wordTable["ham", names(counts.ham)] = counts.ham + .5  
    
    
    # Find the total number of spam and ham
    numSpam = sum(spam)
    numHam = length(spam) - numSpam
    
    # Prob(word | spam) and Prob(word | ham)
    wordTable["spam", ] = wordTable["spam", ]/(numSpam + .5)
    wordTable["ham", ] = wordTable["ham", ]/(numHam + .5)
    
    # log odds
    wordTable["presentLogOdds", ] = 
      log(wordTable["spam",]) - log(wordTable["ham", ])
    wordTable["absentLogOdds", ] = 
      log((1 - wordTable["spam", ])) - log((1 -wordTable["ham", ]))
    
    invisible(wordTable)
  }
trainTable = computeFreqs(trainMsgWords, trainIsSpam)
```

This function helps calculate the log likelihood ratio(LLR) for all of the text messages. And we apply this function to each of the messages in our test set

```r
#calculate the log likelihood ratio for all of the test messages
computeMsgLLR = function(words, freqTable) 
{
  # Discards words not in training data.
  words = words[!is.na(match(words, colnames(freqTable)))]
  
  # Find which words are present
  present = colnames(freqTable) %in% words
  
  sum(freqTable["presentLogOdds", present]) +
    sum(freqTable["absentLogOdds", !present])
}
#apply to all messages
testLLR = sapply(testMsgWords, computeMsgLLR, trainTable)
```

We compare the summarty statistics of the LLR values for the ham and spam in the tset data and we'll draw a boxplot. This log likelihood ratio, log(P(spam | message content)/P(ham | messge comtent)), for test messages was computed using a na??ve Bayes approximation based on word frequencies found in manually classified training data. The test  messages are grouped according to whether they are spam or ham. Notice most ham messages have values well below 0 and nearly all spam values are above 0.

```r
tapply(testLLR, testIsSpam, summary)
```

```
## $`FALSE`
##     Min.  1st Qu.   Median     Mean  3rd Qu.     Max. 
## -1361.00  -127.40  -101.50  -116.60   -81.61   699.80 
## 
## $`TRUE`
##      Min.   1st Qu.    Median      Mean   3rd Qu.      Max. 
##   -60.930     6.549    49.560   138.200   131.400 23610.000
```

```r
spamLab = c("ham", "spam")[1 + testIsSpam]
boxplot(testLLR ~ spamLab, ylab = "Log Likelihood Ratio",
        #  main = "Log Likelihood Ratio for Randomly Chosen Test Messages",
        ylim=c(-500, 500))
```

![](spam_email_files/figure-html/unnamed-chunk-14-1.png) 

###Calculate Type I and Type II error
Now we need to decide on a cut-off where we classify a message as spam or ham according to whether or not the LLR exceeds this threshold. We find the proportion of ham messages in the test set with LLR values that exceed the threshold and so are misclassified as spam. This is exactly the Type I error rate for the test data. We will also find the proportion of LLR values for spam messages in the test set that are below the threshold and so misclassified as ham, which is the Type II error rate.
Our Type I error function only looks at the LLR values for ham messages. It is because only the values corresponding to ham messages will affect the Type I error due to the fact that the messages that are spam do not contribute to the Type I error. The Type II errors function is computed in a similar way.

```r
#efficient way of getting type I by looking at the likelihood rate of ham (a vectorized way)
typeIErrorRates = 
  function(llrVals, isSpam) 
  {
    o = order(llrVals)
    llrVals =  llrVals[o]
    isSpam = isSpam[o]
    
    idx = which(!isSpam)
    N = length(idx)
    list(error = (N:1)/N, values = llrVals[idx])
  }

typeIIErrorRates = function(llrVals, isSpam) {
  
  o = order(llrVals)
  llrVals =  llrVals[o]
  isSpam = isSpam[o]
  
  
  idx = which(isSpam)
  N = length(idx)
  list(error = (1:(N))/N, values = llrVals[idx])
} 
xI = typeIErrorRates(testLLR, testIsSpam)
xII = typeIIErrorRates(testLLR, testIsSpam)
#the tau when type I is 0.01 and the type II given type I is 0.01
tau01 = round(min(xI$values[xI$error <= 0.01]))
t2 = max(xII$error[ xII$values < tau01 ])
```

The Type I and II error rates for the 3116 test messages are shown as a function of the threshold ??. For example, with a threshold of  ?? = -43, all messages with an LLR value above -43 are classified as spam and those below as ham. In this case, 1% of ham is misclassified as spam and 2% of spam is misclassified as ham. Therefore, a threshold of -43 looks reasonable.

```r
library(RColorBrewer)
cols = brewer.pal(9, "Set1")[c(3, 4, 5)]
plot(xII$error ~ xII$values,  type = "l", col = cols[1], lwd = 3,
     xlim = c(-300, 250), ylim = c(0, 1),
     xlab = "Log Likelihood Ratio Values", ylab="Error Rate")
points(xI$error ~ xI$values, type = "l", col = cols[2], lwd = 3)
legend(x = 50, y = 0.4, fill = c(cols[2], cols[1]),
       legend = c("Classify Ham as Spam", 
                  "Classify Spam as Ham"), cex = 0.8,
       bty = "n")
abline(h=0.01, col ="grey", lwd = 3, lty = 2)
text(-250, 0.05, pos = 4, "Type I Error = 0.01", col = cols[2])

mtext(tau01, side = 1, line = 0.5, at = tau01, col = cols[3])
segments(x0 = tau01, y0 = -.50, x1 = tau01, y1 = t2, 
         lwd = 2, col = "grey")
text(tau01 + 20, 0.05, pos = 4,
     paste("Type II Error = ", round(t2, digits = 2)), 
     col = cols[1])
```

![](spam_email_files/figure-html/unnamed-chunk-16-1.png) 

###Cross Validation
The threshold we found may work well with this particular test set but may not work well with others. To address this problem, we can apply method of cross-validation. That is we partition the training data into k parts at random. Then we use each of these parts to act as a test set and compute LLR values for the messages in the subset using the remaining data as a training set. We pool all of these LLR values from all k validation sets to select the threshold ??. In this case, we use k=5 and we find that ??=-33 corresponds to 1% Type I error. We apply this threshold to our original test set and the Type I error is 0.8% and Type II error is 4%.

```r
#use cross validation to get the threshold and the according type I and type II
k = 5
numTrain = length(trainMsgWords)
partK = sample(numTrain)
tot = k * floor(numTrain/k)
partK = matrix(partK[1:tot], ncol = k)

testFoldOdds = NULL
for (i in 1:k) {
  foldIdx = partK[ , i]
  trainTabFold = computeFreqs(trainMsgWords[-foldIdx], trainIsSpam[-foldIdx])
  testFoldOdds = c(testFoldOdds, 
                   sapply(trainMsgWords[ foldIdx ], computeMsgLLR, trainTabFold))
}

testFoldSpam = NULL
for (i in 1:k) {
  foldIdx = partK[ , i]
  testFoldSpam = c(testFoldSpam, trainIsSpam[foldIdx])
}

xFoldI = typeIErrorRates(testFoldOdds, testFoldSpam)
xFoldII = typeIIErrorRates(testFoldOdds, testFoldSpam)
tauFoldI = round(min(xFoldI$values[xFoldI$error <= 0.01]))
tFold2 = xFoldII$error[ xFoldII$values < tauFoldI ]
classify1 = testLLR > tauFoldI;
XI = sum(classify1 & !testIsSpam)/sum(!testIsSpam)
XI
```

```
## [1] 0.007768666
```

```r
classify2 = testLLR < tauFoldI;
XII = 1-sum(classify1 & testIsSpam)/sum(testIsSpam)
XII 
```

```
## [1] 0.03754693
```

###Recursive Partitioning
In order to prepare raw data, we use the method called recursive partitioning which we can create a classifier based on the characteristics of spam. We will split the data into two groups according to the value of a particular variable, and then a subsequent split divides one of these sub-group into two groups according to the value of some variable. This splitting process continue s until the messages are partitioned into subsets that are nearly all spam or ham.
Now that we have a general idea of this approach, we can identify the tasks we need to accomplish to carry it out. We will: 
Process the email into a format that gives easy access to the information in the header, body, and attachments  
Develop functions for the features of interest that transform the email messages into variables for analysis 
Apply to these derived variables the recursive partitioning method and assess how well it predicts spam and ham

###Plans to organize email message into R data structure
In order to store the raw email message in a data structure, we are going to create a list of R objects with one object per message. For each message, it will contain: a named character vector for the header; a character vector with the message content; a data frame with the summary information about the attachments; and a logical indicating if the message is spam or not. The first step will be to split the message into two parts - the header and body.

###Processing the Header
We plan to process the header by converting the key: value pairs into a named vector, where the name of each element in the vector is taken as key. We can use this function to process the header in attachments within the message body.

```r
#process with header
#original header vector as input and named character as output
processHeader = function(header)
{
  # modify the first line to create a key:value pair
  header[1] = sub("^From", "Top-From:", header[1])
  # read lines of the form key
  headerMat = read.dcf(textConnection(header), all = T)
  #close the file after reading
  close(textConnection(header))
  #convert it into a character vector and use the key for the name of each of these values
  headerVec = unlist(headerMat)
  #use the key for the name of each of the values
  dupKeys = sapply(headerMat, function(x) length(unlist(x)))
  names(headerVec) = rep(colnames(headerMat), dupKeys)
  return(headerVec)
  
}
#use the function on sample messages
headerList = lapply(sampleSplit, 
                    function(msg) {
                      processHeader(msg$header)} )
#return content type key
contentTypes = sapply(headerList, function(header) 
  header["Content-Type"])
names(contentTypes) = NULL
contentTypes
```

```
##  [1] "text/plain; charset=us-ascii"                                                                                   
##  [2] "text/plain; charset=US-ASCII"                                                                                   
##  [3] "text/plain; charset=US-ASCII"                                                                                   
##  [4] "text/plain; charset=\"us-ascii\""                                                                               
##  [5] "text/plain; charset=US-ASCII"                                                                                   
##  [6] "multipart/signed;\nboundary=\"==_Exmh_-1317289252P\";\nmicalg=pgp-sha1;\nprotocol=\"application/pgp-signature\""
##  [7] NA                                                                                                               
##  [8] "multipart/alternative;\nboundary=\"----=_NextPart_000_00C1_01C25017.F2F04E20\""                                 
##  [9] "multipart/alternative; boundary=Apple-Mail-2-874629474"                                                         
## [10] "multipart/signed;\nboundary=\"==_Exmh_-518574644P\";\nmicalg=pgp-sha1;\nprotocol=\"application/pgp-signature\"" 
## [11] "multipart/related;\nboundary=\"------------090602010909000705010009\""                                          
## [12] "multipart/signed;\nboundary=\"==_Exmh_-451422450P\";\nmicalg=pgp-sha1;\nprotocol=\"application/pgp-signature\"" 
## [13] "multipart/signed;\nboundary=\"==_Exmh_267413022P\";\nmicalg=pgp-sha1;\nprotocol=\"application/pgp-signature\""  
## [14] "multipart/mixed;\nboundary=\"----=_NextPart_000_0005_01C26412.7545C1D0\""                                       
## [15] "multipart/alternative;\nboundary=\"------------080209060700030309080805\""
```

###Processing Attachments
Now we want to extract the attachments from the body and summarize them. We use the boundary string to locate the attachments in the body. We will store the information including the attachment's MIME type and its length. We also need to be aware of some messages which may not have an attachment even though their header indicates that they are supposed to. We will:
Drop the lines before the first boundary string, which marks the first part of the body
Keep the lines following the closing string as part of the body without attachments
Include the header information in the line count for the attachments
Use the last line of the email as the end of the attachment if we find no closing boundary string

```r
#processing attachments
processAttach = function(body, contentType){
  
  n = length(body)
  boundary = getBoundary(contentType)
  #locate the attachments by searching for the boundary string preceeded by 2 hyphens in the body
  bString = paste("--", boundary, sep = "")
  bStringLocs = which(bString == body)
  #find the closing boundary
  eString = paste("--", boundary, "--", sep = "")
  eStringLoc = which(eString == body)
  #use the last line as the end of attachment
  if (length(eStringLoc) == 0) eStringLoc = n
  #the case no attachment
  if (length(bStringLocs) <= 1) {
    attachLocs = NULL
    msgLastLine = n
    if (length(bStringLocs) == 0) bStringLocs = 0
  } else {
    #do have attachment
    attachLocs = c(bStringLocs[ -1 ],  eStringLoc)
    msgLastLine = bStringLocs[2] - 1
  }
  #extract the message
  msg = body[ (bStringLocs[1] + 1) : msgLastLine] 
  #the case that the attachment is in the middle
  if ( eStringLoc < n )
    msg = c(msg, body[ (eStringLoc + 1) : n ])
  #if there's an attachment
  if ( !is.null(attachLocs) ) {
    #the length of attachment
    attachLens = diff(attachLocs, lag = 1) 
    #the type of attachment
    attachTypes = mapply(function(begL, endL) {
      #find content-type
      CTloc = grep("^[Cc]ontent-[Tt]ype", body[ (begL + 1) : (endL - 1)])
      if ( length(CTloc) == 0 ) {
        MIMEType = NA
      } else {
        CTval = body[ begL + CTloc[1] ]
        CTval = gsub('"', "", CTval )
        MIMEType = sub(" *[Cc]ontent-[Tt]ype: *([^;]*);?.*", "\\1", CTval)   
      }
      return(MIMEType)
    }, attachLocs[-length(attachLocs)], attachLocs[-1])
  }
 #output the results
  #2 elements: the first is the body containing the message and no attachments; the second is a data frame called attachDF
  # which contains the length and type of each attachment
  if (is.null(attachLocs)) return(list(body = msg, attachDF = NULL) )
  return(list(body = msg, 
              attachDF = data.frame(aLen = attachLens, 
                                    aType = unlist(attachTypes),
                                    stringsAsFactors = FALSE)))                                
} 
#applied on sample emails
bodyList = lapply(sampleSplit, function(msg) msg$body)
attList = mapply(processAttach, bodyList[hasAttach], 
                 contentTypes[hasAttach], 
                 SIMPLIFY = FALSE)
```

After completing and testing all of the tasks to transform one message, we use this function to apply all the tasks to all the emails.

```r
#read email
readEmail = function(dirName) {
  # retrieve the names of files in directory
  fileNames = list.files(dirName, full.names = TRUE)
  # drop files that are not email
  notEmail = grep("cmds$", fileNames)
  if ( length(notEmail) > 0) fileNames = fileNames[ - notEmail ]
  
  # read all files in the directory
  lapply(fileNames, readLines, encoding = "latin1")
}
```

This function pulls together various tasks.

```r
processAllEmail = function(dirName, isSpam = FALSE)
{
  # read all files in the directory
  messages = readEmail(dirName)
  fileNames = names(messages)
  n = length(messages)
  
  # split header from body
  eSplit = lapply(messages, splitMessage)
  rm(messages)
  
  # process header as named character vector
  headerList = lapply(eSplit, function(msg) 
    processHeader(msg$header))
  
  # extract content-type key
  contentTypes = sapply(headerList, function(header) 
    header["Content-Type"])
  
  # extract the body
  bodyList = lapply(eSplit, function(msg) msg$body)
  rm(eSplit)
  
  # which email have attachments
  hasAttach = grep("^ *multi", tolower(contentTypes))
  
  # get summary stats for attachments and the shorter body
  attList = mapply(processAttach, bodyList[hasAttach], 
                   contentTypes[hasAttach], SIMPLIFY = FALSE)
  
  bodyList[hasAttach] = lapply(attList, function(attEl) 
    attEl$body)
  
  attachInfo = vector("list", length = n )
  attachInfo[ hasAttach ] = lapply(attList, 
                                   function(attEl) attEl$attachDF)
  
  # prepare return structure
  emailList = mapply(function(header, body, attach, isSpam) {
    list(isSpam = isSpam, header = header, 
         body = body, attach = attach)
  },
  headerList, bodyList, attachInfo, 
  rep(isSpam, n), SIMPLIFY = FALSE )
  names(emailList) = fileNames
  
  invisible(emailList)
}
```

We apply the function to each directory and extract the same set of sample email messages that we used in developing processAllEmail(). This will help us develop the functions to create the feature set.  

```r
emailStruct = mapply(processAllEmail, fullDirNames,
                     isSpam = rep( c(FALSE, TRUE), 3:2))     
```

```
## Warning in FUN(X[[i]], ...): incomplete final line found on '/Users/
## Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/hard_ham/
## 00228.0eaef7857bbbf3ebf5edbbdae2b30493'
```

```
## Warning in FUN(X[[i]], ...): incomplete final line found on '/Users/
## Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/hard_ham/
## 0231.7c6cc716ce3f3bfad7130dd3c8d7b072'
```

```
## Warning in FUN(X[[i]], ...): incomplete final line found on '/Users/
## Yifan/Documents/MASDA/Data science/final/SpamAssassinTrain/hard_ham/
## 0250.7c6cc716ce3f3bfad7130dd3c8d7b072'
```

```r
emailStruct = unlist(emailStruct, recursive = FALSE)
sampleStruct = emailStruct[ indx ]
save(emailStruct, file="emailXX.rda")
```

We now focus our attention to the use of capitalization. The over use of capitalization in email is referred as yelling and messages that yell a lot tend to be spam. We will take several approaches to quantify the amount of yelling in a message. We may:
Look at the subject line in the header and determine whether it is all capitals or not
Report the percentage of capital letters among all letters used in the body
We will create a list of functions and we can apply this list of functions to our email structure to make a data frame of variables. Then it will make it easier for us to apply them to our email corpus.

```r
#derive variables from email message
funcList = list( 
  #check whether Re: appears at the start of the subject
  isRe = function(msg) {
    "Subject" %in% names(msg$header) &&
      length(grep("^[ \t]*Re:", msg$header[["Subject"]])) > 0
  },
  #get the number of lines
  numLines = function(msg) 
    length(msg$body),
  #whether all of the alpha characters in the subject line are upper case with NA indicating no subject
  isYelling = function(msg) {
    if ( "Subject" %in% names(msg$header) ) {
      #eliminate numbers and blanks in the header
      el = gsub("[^[:alpha:]]", "", msg$header["Subject"])
      if (nchar(el) > 0) 
        #if is blank after all the captial letters are taken away
        nchar(gsub("[A-Z]", "", el)) < 1
      else 
        FALSE
    }
    #no subject
    else NA
  },
  #get the percentage of capitalized letters in the message body 
  perCaps = function(msg) {
    body = paste(msg$body, collapse = "")
    
    # Return NA if the body of the message is "empty"
    if(length(body) == 0 || nchar(body) == 0) return(NA)
    
    # Eliminate non-alpha characters
    body = gsub("[^[:alpha:]]", "", body)
    capText = gsub("[^A-Z]", "", body)
    # 100*number of upper case letters/number of upper and lower case letters
    100 * nchar(capText)/nchar(body)
  }
)
```

Allow the list provided in the operations argument to include expressions as well as functions. The operation is applied to the email slightly differently. For an expression, we evaluate the expression in an environment which contains the variable "msg". This is the individual message object to be processed. The expression must refer to that variable directly. The benefit of an expression is it???s simpler to write than a function but harder to reason about how it will be evaluated and where the variables to which it refers are located.

```r
createDerivedDF =
  function(email = emailStruct, operations = funcList, 
           verbose = FALSE)
  {
    els = lapply(names(operations),
                 function(id) {
                   if(verbose) print(id)
                   e = operations[[id]]
                   v = if(is.function(e)) 
                     sapply(email, e)
                   else 
                     #evaluate the expression which contains variable msg
                     sapply(email, function(msg) eval(e))
                   v
                 })
    
    df = as.data.frame(els)
    names(df) = names(operations)
    invisible(df)
  }
```

Now we write a function which derives the remaining variables which might be promising as possible predictors of spam or ham. It will include the functions in the previous function list. We will also give a vector of spam-check-words for the use of some functions.

```r
SpamCheckWords =
  c("viagra", "pounds", "free", "weight", "guarantee", "million", 
    "dollars", "credit", "risk", "prescription", "generic", "drug",
    "financial", "save", "dollar", "erotic", "million", "barrister",
    "beneficiary", "easy", 
    "money back", "money", "credit card")

getMessageRecipients =
  function(header)
  {
    c(if("To" %in% names(header))  header[["To"]] else character(0),
      if("Cc" %in% names(header))  header[["Cc"]] else character(0),
      if("Bcc" %in% names(header)) header[["Bcc"]] else character(0)
    )
  }

funcList = list(
  #is spam
  isSpam =
    expression(msg$isSpam)
  ,
  #Type: logical,   TRUE if Re: appears at the start of the subject
  isRe =
    function(msg) {
      # Can have a Fwd: Re:  ... but we are not looking for this here.
      # We may want to look at In-Reply-To field.
      "Subject" %in% names(msg$header) && 
        length(grep("^[ \t]*Re:", msg$header[["Subject"]])) > 0
    }
  ,
  #Type: integer,  Number of lines in the body of the message
  numLines =
    function(msg) length(msg$body)
  ,
  #Type: integer, number of characters in in the body of the message
  bodyCharCt =
    function(msg)
      sum(nchar(msg$body))
  ,
  #Type: logical,  TRUE if email address in the From field of the header contains an underscore
  underscore =
    function(msg) {
      if(!"Reply-To" %in% names(msg$header))
        return(FALSE)
      
      txt <- msg$header[["Reply-To"]]
      length(grep("_", txt)) > 0  && 
        length(grep("[0-9A-Za-z]+", txt)) > 0
    }
  ,
  #Type: integer, check the number of exclamatory mark in subject
  subExcCt = 
    function(msg) {
      x = msg$header["Subject"]
      if(length(x) == 0 || sum(nchar(x)) == 0 || is.na(x))
        return(NA)
      
      sum(nchar(gsub("[^!]","", x)))
    }
  ,
  #Type: integer, check the number of question mark in subject
  subQuesCt =
    function(msg) {
      x = msg$header["Subject"]
      if(length(x) == 0 || sum(nchar(x)) == 0 || is.na(x))
        return(NA)
      
      sum(nchar(gsub("[^?]","", x)))
    }
  ,
  #Type: integer, number of attachment in the message
  numAtt = 
    function(msg) {
      if (is.null(msg$attach)) return(0)
      else nrow(msg$attach)
    }
  ,
  # Type: logical, TRUE if a Priority key is present in the header
  priority =
    function(msg) {
      ans <- FALSE
      # Look for names X-Priority, Priority, X-Msmail-Priority
      # Look for high any where in the value
      ind = grep("priority", tolower(names(msg$header)))
      if (length(ind) > 0)  {
        ans <- length(grep("high", tolower(msg$header[ind]))) >0
      }
      ans
    }
  ,
  #Type: numeric, check the number of recipients of the message, including CCs
  numRec =
    function(msg) {
      # unique or not.
      els = getMessageRecipients(msg$header)
      
      if(length(els) == 0)
        return(NA)
      # Split each line by ","  and in each of these elements, look for
      # the @ sign. This handles
      tmp = sapply(strsplit(els, ","), function(x) grep("@", x))
      sum(sapply(tmp, length))
    }
  ,
  #Type: numeric, percentage of capitalized letter among all letter in body, excluding attachments
  perCaps =
    function(msg)
    {
      body = paste(msg$body, collapse = "")
      
      # Return NA if the body of the message is "empty"
      if(length(body) == 0 || nchar(body) == 0) return(NA)
      
      # Eliminate non-alpha characters and empty lines 
      body = gsub("[^[:alpha:]]", "", body)
      els = unlist(strsplit(body, ""))
      ctCap = sum(els %in% LETTERS)
      100 * ctCap / length(els)
    }
  ,
  #Type: logical, TRUE if the In-Reply-To key is present in the header 
  isInReplyTo =
    function(msg)
    {
      "In-Reply-To" %in% names(msg$header)
    }
  ,
  #Type: logical, TRUE if the recipients??? email addresses are sorted
  sortedRec =
    function(msg)
    {
      ids = getMessageRecipients(msg$header)
      all(sort(ids) == ids)
    }
  ,
  #Type: logical, TRUE if words in the subject have punctuation or numbers embedded in them
  subPunc =
    function(msg)
    {
      if("Subject" %in% names(msg$header)) {
        #get rid of punctuations 
        el = gsub("['/.:@-]", "", msg$header["Subject"])
        length(grep("[A-Za-z][[:punct:]]+[A-Za-z]", el)) > 0
      }
      else
        FALSE
    },
  #Type: numeric, Hour of the day in the Data field
  hour =
    function(msg)
    {
      #look for date
      date = msg$header["Date"]
      if ( is.null(date) ) return(NA)
      # Need to handle that there may be only one digit in the hour
      locate = regexpr("[0-2]?[0-9]:[0-5][0-9]:[0-5][0-9]", date)
      
      if (locate < 0)
        locate = regexpr("[0-2]?[0-9]:[0-5][0-9]", date)
      if (locate < 0) return(NA)
      
      hour = substring(date, locate, locate+1)
      hour = as.numeric(gsub(":", "", hour))
      #turn into 24 hours a day
      locate = regexpr("PM", date)
      if (locate > 0) hour = hour + 12
      locate = regexpr("[+-][0-2][0-9]00", date)
      #whether move a day forward or not
      if (locate < 0) offset = 0
      else offset = as.numeric(substring(date, locate, locate + 2))
      (hour - offset) %% 24
    }
  ,
  #Type: logical, TRUE if the MIME type is multipart/text
  multipartText =
    function(msg)
    {
      if (is.null(msg$attach)) return(FALSE)
      #number of attachment
      numAtt = nrow(msg$attach)
      types = 
        length(grep("(html|plain|text)", msg$attach$aType)) > (numAtt/2)
    }
  ,
  #Type: logical, True if it has images inside the message
  hasImages =
    function(msg)
    {
      if (is.null(msg$attach)) return(FALSE)
      
      length(grep("^ *image", tolower(msg$attach$aType))) > 0
    }
  ,
  #Type: logical, True if it has pgp signature inside message
  isPGPsigned =
    function(msg)
    {
      if (is.null(msg$attach)) return(FALSE)
      
      length(grep("pgp", tolower(msg$attach$aType))) > 0
    },
  #Type: numeric, Percentage of characters in HTML tags in the message body in comparison to all characters
  perHTML =
    function(msg)
    {
      if(! ("Content-Type" %in% names(msg$header))) return(0)
      
      el = tolower(msg$header["Content-Type"]) 
      #if no html
      if (length(grep("html", el)) == 0) return(0)
      #remove space
      els = gsub("[[:space:]]", "", msg$body)
      totchar = sum(nchar(els))
      #remove all the elements not starting with <
      totplain = sum(nchar(gsub("<[^<]+>", "", els )))
      100 * (totchar - totplain)/totchar
    },
  #Type: logical, TRUE if the subject contains one of the words in a spam word vector
  subSpamWords =
    function(msg)
    {
      if("Subject" %in% names(msg$header))
        length(grep(paste(SpamCheckWords, collapse = "|"), 
                    tolower(msg$header["Subject"]))) > 0
      else
        NA
    }
  ,
  #Type: numeric, Percentage of blanks in the subject
  subBlanks =
    function(msg)
    {
      if("Subject" %in% names(msg$header)) {
        x = msg$header["Subject"]
        # should we count blank subject line as 0 or 1 or NA?
        if (nchar(x) == 1) return(0)
        else 100 *(1 - (nchar(gsub("[[:blank:]]", "", x))/nchar(x)))
      } else NA
    }
  ,
  #Type: logical, TRUE if there is no hostname in the Message-Id key in the header
  noHost =
    function(msg)
    {
      # Or use partial matching.
      idx = pmatch("Message-", names(msg$header))
      
      if(is.na(idx)) return(NA)
      
      tmp = msg$header[idx]
      return(length(grep(".*@[^[:space:]]+", tmp)) ==  0)
    }
  ,
  #Type: logical, TRUE if the email sender???s address (before the @) ends in a number
  numEnd =
    function(msg)
    {
      # If we just do a grep("[0-9]@",  )
      # we get matches on messages that have a From something like
      # " \"marty66@aol.com\" <synjan@ecis.com>"
      # and the marty66 is the "user's name" not the login
      # So we can be more precise if we want.
      x = names(msg$header)
      if ( !( "From" %in% x) ) return(NA)
      login = gsub("^.*<", "", msg$header["From"])
      if ( is.null(login) ) 
        login = gsub("^.*<", "", msg$header["X-From"])
      if ( is.null(login) ) return(NA)
      login = strsplit(login, "@")[[1]][1]
      length(grep("[0-9]+$", login)) > 0
    },
  #Type: logical, TRUE if the subject is all capital letters
  isYelling =
    function(msg)
    {
      if ( "Subject" %in% names(msg$header) ) {
        el = gsub("[^[:alpha:]]", "", msg$header["Subject"])
        if (nchar(el) > 0) nchar(gsub("[A-Z]", "", el)) < 1
        else FALSE
      }
      else
        NA
    },
  #Type: numeric, Number if forward symbols in a line of the body
  forwards =
    function(msg)
    {
      x = msg$body
      if(length(x) == 0 || sum(nchar(x)) == 0)
        return(NA)
      
      ans = length(grep("^[[:space:]]*>", x))
      100 * ans / length(x)
    },
  #Type: logical, TRUE if the message body contains the phrase original message
  isOrigMsg =
    function(msg)
    {
      x = msg$body
      if(length(x) == 0) return(NA)
      
      length(grep("^[^[:alpha:]]*original[^[:alpha:]]+message[^[:alpha:]]*$", 
                  tolower(x) ) ) > 0
    },
  #Type: logical, TRUE if the message body contains the word dear
  isDear =
    function(msg)
    {
      x = msg$body
      if(length(x) == 0) return(NA)
      
      length(grep("^[[:blank:]]*dear +(sir|madam)\\>", 
                  tolower(x))) > 0
    }
  ,
  #Type: logical, TRUE if the message contains the phrase wrote|schrieb|ecrit|escribe
  isWrote =
    function(msg)
    {
      x = msg$body
      if(length(x) == 0) return(NA)
      
      length(grep("(wrote|schrieb|ecrit|escribe):", tolower(x) )) > 0
    }
  ,
  #Type: numeric, The average length of the words in a message
  avgWordLen =
    function(msg)
    {
      txt = paste(msg$body, collapse = " ")
      if(length(txt) == 0 || sum(nchar(txt)) == 0) return(0)
      #remove not letter things
      txt = gsub("[^[:alpha:]]", " ", txt)
      words = unlist(strsplit(txt, "[[:blank:]]+"))
      wordLens = nchar(words)
      mean(wordLens[ wordLens > 0 ])
    }
  ,
  ##Type: numeric, Number of dollar sign in the message body
  numDlr =
    function(msg)
    {
      x = paste(msg$body, collapse = "")
      if(length(x) == 0 || sum(nchar(x)) == 0)
        return(NA)
      nchar(gsub("[^$]","", x))
    }
)
```

The data frame emailDF in RSpamData contains the conversion of the complete set of 9348 email into the 29 variables.

```r
#use all the functions on dataset
emailDF = createDerivedDF(emailStruct)
dim(emailDF)
```

```
## [1] 9348   30
```

This scatter plot shows the relationship between the number of lines and the number of characters in the body of a message. The plot is on log scale, and 1 is added to all of the values before taking logs to address issues with empty bodies. The line y=x is added for comparison purposes. As might be expected, they appear to have a highly linear association.

```r
#scatterplot of number of lines against number of characters
x.at = c(1,10,100,1000,10000,100000)
y.at = c(1, 5, 10, 50, 100, 500, 5000)
nL = 1 + emailDF$numLines
nC = 1 + emailDF$bodyCharCt
plot(nL ~ nC, log = "xy", pch=".", xlim=c(1,100000), axes = FALSE,
     xlab = "Number of Characters", ylab = "Number of Lines")
box() 
axis(1, at = x.at, labels = formatC(x.at, digits = 0, format="d"))
axis(2, at = y.at, labels = formatC(y.at, digits = 0, format="d")) 
abline(a=0, b=1, col="red", lwd = 2)
```

![](spam_email_files/figure-html/unnamed-chunk-27-1.png) 

Now let's examine the percentage of capitals in the message body, comparing the percentage between spam and ham messages. We draw side-by-side boxplots to compare the distribution of these two groups. The use of a log scale makes it easier to see that nearly 3/4 of the spam have more capital letters than nearly all of the ham, which makes this variable be useful for classification.

```r
#boxplot of percent of capitals
#pdf(paste(getwd(),"/SPAM_boxplotsPercentCaps.pdf"), width = 5, height = 5)
percent = emailDF$perCaps
isSpamLabs = factor(emailDF$isSpam, labels = c("ham", "spam"))
boxplot(log(1 + percent) ~ isSpamLabs,
        ylab = "Percent Capitals (log)")
```

![](spam_email_files/figure-html/unnamed-chunk-28-1.png) 

```r
dev.off()
```

```
## null device 
##           1
```

QQ plots also indicate that spam email have higher percentage of capital letters than regular email. The spam messages have a larger average number of capital letters and a greater spread than non-spam messages.

```r
#percentage of capital letters for two types of emails
logPerCapsSpam = log(1 + emailDF$perCaps[ emailDF$isSpam ])
logPerCapsHam = log(1 + emailDF$perCaps[ !emailDF$isSpam ])
qqplot(logPerCapsSpam, logPerCapsHam, 
       xlab = "Regular Email", ylab = "Spam Email", 
       main = "Percentage of Capital Letters (log scale)",
       pch = 19, cex = 0.3)
```

![](spam_email_files/figure-html/unnamed-chunk-29-1.png) 

We can compare the joint distribution of the percentage of capital letters in the email and the total number of characters in the body of the message for spam vs. ham messages. This scatter plot examines the relationship between the percentage of capital letters among all letters in message and the total number of characters in the message. Spam is marked by purple dots and ham by green. The darker color indicates over plotting. We see here that the spam tends to be longer and have more capital letters than ham.

```r
colI = c("#4DAF4A80", "#984EA380")
logBodyCharCt = log(1 + emailDF$bodyCharCt)
logPerCaps = log(1 + emailDF$perCaps)
plot(logPerCaps ~ logBodyCharCt, xlab = "Total Characters (log)",
     ylab = "Percent Capitals (log)",
     col = colI[1 + emailDF$isSpam],
     xlim = c(2,12), pch = 19, cex = 0.5)
```

![](spam_email_files/figure-html/unnamed-chunk-30-1.png) 

Exploring Categorical Measures Derived from email. These two mosaic plots use area to denote the proportion of messages that fall in each category. The plot on the top shows those messages that have an Re: in the subject line tend not to be spam. The bottom plot shows that those messages that are from a user with a number at the end of their email address tend to be spam. However, few messages are sent from such users so it is not clear how helpful distinction will be in our classification problem. 

```r
oldPar = par(mfrow = c(1, 2), mar = c(1,1,1,1))
colM = c("#E41A1C80", "#377EB880")
isRe = factor(emailDF$isRe, labels = c("no Re:", "Re:"))
mosaicplot(table(isSpamLabs, isRe), main = "",
           xlab = "", ylab = "", color = colM)
fromNE = factor(emailDF$numEnd, labels = c("No #", "#"))
mosaicplot(table(isSpamLabs, fromNE), color = colM,
           main = "", xlab="", ylab = "")
```

![](spam_email_files/figure-html/unnamed-chunk-31-1.png) 

```r
par(oldPar)
```

###Recursive partitioning
For now, we proceed to apply the rpart() method to the 29 variables in emailDF and assess how well they can classify email. First of all, we must convert the logical to factors because the variables in the data frame must all be either factors or numeric.

```r
#decision tree
library(rpart)
setupRpart = function(data) {
  #find logical variables
  logicalVars = which(sapply(data, is.logical))
  facVars = lapply(data[ , logicalVars], 
                   function(x) {
                     x = as.factor(x)
                     levels(x) = c("F", "T")
                     x
                   })
  cbind(facVars, data[ , - logicalVars])
}
emailDFrp = setupRpart(emailDF)
```

Now that our data are properly formatted, we split them into training and test sets as we did in the naive Bayes analysis. We use the same subsets as with the text mining so we can accurately compare these 2 approaches. Then we fit the classification tree.

```r
#set test data
set.seed(418910)
testSpamIdx = sample(numSpam, size = floor(numSpam/3))
testHamIdx = sample(numHam, size = floor(numHam/3))
testDF = 
  rbind( emailDFrp[ emailDFrp$isSpam == "T", ][testSpamIdx, ],
         emailDFrp[emailDFrp$isSpam == "F", ][testHamIdx, ] )
trainDF =
  rbind( emailDFrp[emailDFrp$isSpam == "T", ][-testSpamIdx, ], 
         emailDFrp[emailDFrp$isSpam == "F", ][-testHamIdx, ])

rpartFit = rpart(isSpam ~ ., data = trainDF, method = "class")
```

We plot the fitted tree with the following code and the resulting tree is given below. This tree was fitted using rpart() on 6232 messages. The default values for all of the arguments to rpart() were used. Notice the leftmost leaf classifies as ham those messages with fewer than 13% capitals, fewer than 4% HTML tags, and at least 1 forward. Eighteen spam messages fall into this leaf and so are misclassified, but 2240 the ham is properly classified using these 3 yes-no questions.

```r
library(rpart.plot)
prp(rpartFit, extra = 1)
```

![](spam_email_files/figure-html/unnamed-chunk-34-1.png) 

Now let's see how well the classifier does at predicting whether the test messages are spam or ham. To compare how well the tree has performed in classifying test messages, we compare the predictions from the fitted rpart object to the hand classifications. We'll find Type I error, the proportion of ham messages that have been misclassified as spam. We'll also find Type II error, the proportion of spam messages that have been misclassified as ham.

```r
#use the decision tree to do prediction
predictions = predict(rpartFit, 
                      newdata = testDF[, names(testDF) != "isSpam"],
                      type = "class")

#check the result of prediction
predsForHam = predictions[ testDF$isSpam == "F" ]
summary(predsForHam)
```

```
##    F    T 
## 2206  111
```

```r
#typr I
sum(predsForHam == "T") / length(predsForHam)
```

```
## [1] 0.04790678
```

```r
#type II
predsForSpam = predictions[ testDF$isSpam == "T" ]
sum(predsForSpam == "F") / length(predsForSpam)
```

```
## [1] 0.1614518
```

Our classifier did reasonably well with Type I errors, but the Type II error rate is 16%. We will use the complexity parameter, which is used as a threshold where any split that does not decrease the overall lack of fit by cp is not considered. We'll examine how the fit changes when trying different values for cp. We'll also call rpart() with each of these values for cp and ues the resulting model to classify the test data. We finally will assess the Type I and II errors for these fitted models. 

```r
complexityVals = c(seq(0.00001, 0.0001, length=19),
                   seq(0.0001, 0.001, length=19), 
                   seq(0.001, 0.005, length=9),
                   seq(0.005, 0.01, length=9))

fits = lapply(complexityVals, function(x) {
  rpartObj = rpart(isSpam ~ ., data = trainDF,
                   method="class", 
                   control = rpart.control(cp=x) )
  
  predict(rpartObj, 
          newdata = testDF[ , names(testDF) != "isSpam"],
          type = "class")
})

spam = testDF$isSpam == "T"
numSpam = sum(spam)
numHam = sum(!spam)
errs = sapply(fits, function(preds) {
  typeI = sum(preds[ !spam ] == "T") / numHam
  typeII = sum(preds[ spam ] == "F") / numSpam
  c(typeI = typeI, typeII = typeII)
})
```

The errors are displayed in the following plot. We see there that the smallest Type I error that we are able to achieve is about 0.04, which occurs for a complexity value of about 0.001. The Type II error for this complexity value is 10.5%. The text mining approach has smaller Type I and Type II errors. 

```r
library(RColorBrewer)
cols = brewer.pal(9, "Set1")[c(3, 4, 5)]
plot(errs[1,] ~ complexityVals, type="l", col=cols[2], 
     lwd = 2, ylim = c(0,0.2), xlim = c(0,0.005), 
     ylab="Error", xlab="complexity parameter values")
points(errs[2,] ~ complexityVals, type="l", col=cols[1], lwd = 2)

text(x =c(0.003, 0.0035), y = c(0.12, 0.05), 
     labels=c("Type II Error", "Type I Error"))

minI = which(errs[1,] == min(errs[1,]))[1]
abline(v = complexityVals[minI], col ="grey", lty =3, lwd=2)

text(0.0007, errs[1, minI]+0.01, 
     formatC(errs[1, minI], digits = 2))
text(0.0007, errs[2, minI]+0.01, 
     formatC(errs[2, minI], digits = 3))
```

![](spam_email_files/figure-html/unnamed-chunk-37-1.png) 

The poorer performance of recursive partitioning may be due to the variables that were used in the model. Or, it may be due to the parameter settings in rpart(). Additionally, we will try to create a combined approach that employs both vectors and derived variables in predicting spam.
