---
title: "Basic NN for the Avito Demand Prediction Competition on Kaggle"
date: "2018-28-06T22:12:03.284Z"
slug: "basic-neuralnet-avito-demand-prediction-competition-kaggle"
template: "post"
draft: false
category: "Kaggle"
tags:
  - "kaggle"
  - "deep-learning"
description: "I wanted to try building a model for the Avito Demand Prediction competition on Kaggle and I came across a few hurdles on the road, which taught me lots of new things. I wasn't after a good score but I just wanted to build a neural network with Keras from end to end and get predictions from it."
socialImage: "/media/kaggle/thumbnail-kaggle.png"
---

I wanted to try building a model for the Avito Demand Prediction competition on Kaggle and I came across a few hurdles on the road, which taught me lots of new things. I wasn't after a good score but I just wanted to build a neural network with Keras from end to end and get predictions from it.

So for this competition, we're trying to predict demand for an online advertisement based on its full description (title, description, images, etc.) and its context.

Roughly, I wanted to make use of the categorical features and a few continuous features. I also wanted to the features of the images and for this I wanted to get their extracted features from VGG16.

## Getting image features from VGG16
While I was trying to get the image feature extraction through VGG16, I found out that someone else on Kaggle has a kernel for that, so I added his two kernels' outputs (one outputting the VGG16 features of the training data set images and the other one, the test data set images) as the data source of my own kernel and I could use these sparse output matrices within my script.

So I dropped the image column of the datasets and appended these 512 features taken from VGG16, to my train and test data frames (during feeding my model batch by batch, as explained below, because the dense array version of these sparse data was taking lots of memory).

## Cleaning, imputing, dropping, transforming, encoding, scaling
I dropped the columns I won't use and imputed the datasets by filling in the NAN values accordingly. Then I transformed some features like the activation date to weekday and title and description (which were in Russian) to their word counts (I could have dealt with this information by extracting the text features through another established model, but I didn't focus on that).

I encoded the categorical columns with LabelEncoders and I scaled my continuous features between 0 and 1, using the StandardScaler.

## Fitting all the data into 15GB memory
Of course, as a noob, I took my chances to see if I can feed all of this data at once to my model. Apparently and indeed not surprisingly, this wasn't possible with the Kaggle kernels' 15 GB memory limit. Then I learnt about the fit_generator method of Keras, which lets you feed your data to your model, spoon by spoon :] I'm pretty happy that I learnt about this method - hurdles along the way are the best teachers I think.

## Getting NAN loss during training
During the initial steps, my loss value was decreasing but then I started getting NAN loss, before the 1st epoch completed. This made me suspicious of my input data and I found out that I forgot to scale some of my continuous features. I did that and these NAN values were gone.

## Final thoughts

So this pretty basic model didn't do super great in the competition but I at least succeeded in building an end to end model, getting its predictions and I learnt a lot from these small to medium scale hurdles :]
To improve this, I can think of what new features I can introduce with "feature engineering", modify my hyper-parameters or my network architecture or make use of some more features like the description for instance.

So here is my script but of course it won't run properly unless you run it on a Kaggle kernel by adding the necessary datasets. If you spot any mistakes, you are more than welcome to point those out and let me learn from you!


```python
import numpy as np
import pandas as pd
from scipy import sparse
from sklearn.preprocessing import StandardScaler
from sklearn.preprocessing import LabelEncoder
from keras.models import Model
from keras.callbacks import EarlyStopping, ModelCheckpoint
from keras.layers import Embedding, Dense, Input, concatenate, Flatten, Dropout, BatchNormalization
from keras.optimizers import adam
from keras import losses
from keras import metrics


def load_data():
	train_data = pd.read_csv("../input/avito-demand-prediction/train.csv", parse_dates=["activation_date"])
	test_data = pd.read_csv("../input/avito-demand-prediction/test.csv", parse_dates=["activation_date"])

	train_data['activation_date'] = np.expand_dims(
		pd.to_datetime(train_data['activation_date']).dt.weekday.astype(np.int32).values, axis=-1)
	test_data['activation_date'] = np.expand_dims(
		pd.to_datetime(test_data['activation_date']).dt.weekday.astype(np.int32).values, axis=-1)

	y_col = 'deal_probability'

	return train_data, test_data, y_col


def fill_na(train_data, test_data):
	train_data['image_top_1'].fillna(value=3067, inplace=True)
	test_data['image_top_1'].fillna(value=3067, inplace=True)

	train_data['item_seq_number'].fillna(value=-1, inplace=True)
	test_data['item_seq_number'].fillna(value=-1, inplace=True)

	train_data['price'].fillna(value=-1, inplace=True)
	test_data['price'].fillna(value=-1, inplace=True)

	train_data['param_1'].fillna(value='_NA_', inplace=True)
	test_data['param_1'].fillna(value='_NA_', inplace=True)

	train_data['param_2'].fillna(value='_NA_', inplace=True)
	test_data['param_2'].fillna(value='_NA_', inplace=True)

	train_data['param_3'].fillna(value='_NA_', inplace=True)
	test_data['param_3'].fillna(value='_NA_', inplace=True)
	return train_data, test_data


def drop_unwanted(train_data, test_data):
	train_data.drop(['item_id', 'user_id', 'image'], axis=1, inplace=True)
	test_data.drop(['item_id', 'user_id', 'image'], axis=1, inplace=True)
	return train_data, test_data


def encode_cat_columns(all_data):
	cat_cols = ['region', 'parent_category_name', 'category_name', 'city', 'user_type', 'image_top_1', 'param_1',
	            'param_2', 'param_3']

	le_encoders = {x: LabelEncoder() for x in cat_cols}
	label_enc_cols = {k: v.fit_transform(all_data[k]) for k, v in le_encoders.items()}

	return le_encoders, label_enc_cols


def transform_title_description(train_data, test_data):
	train_data['title'] = train_data.title.apply(lambda x: len(str(x).split(' ')))
	train_data['description'] = train_data.description.apply(lambda x: len(str(x).split(' ')))

	test_data['title'] = test_data.title.apply(lambda x: len(str(x).split(' ')))
	test_data['description'] = test_data.description.apply(lambda x: len(str(x).split(' ')))
	return train_data, test_data


def scale_num_cols(train_data, test_data):
	stdScaler = StandardScaler()
	train_data[['price', 'item_seq_number', 'title', 'description']] = stdScaler.fit_transform(train_data[['price', 'item_seq_number', 'title', 'description']])
	test_data[['price', 'item_seq_number', 'title', 'description']] = stdScaler.fit_transform(test_data[['price', 'item_seq_number', 'title', 'description']])
	return train_data, test_data


def load_VGG16_img_features():
	train_img_features = sparse.load_npz('../input/vgg16-train-features/features.npz')
	test_img_features = sparse.load_npz('../input/vgg16-test-features/features.npz')
	return train_img_features, test_img_features


def split_train_validation(train_data):
	val_split = 0.15
	val_ix = int(np.rint(len(train_data) * (1. - val_split)))

	t_split_df = train_data[:val_ix]
	v_split_df = train_data[val_ix:]

	image_t_split_df = train_img_features[:val_ix]
	image_v_split_df = train_img_features[val_ix:]

	return t_split_df, v_split_df, image_t_split_df, image_v_split_df


def gen_samples(in_df, img_df, batch_size, loss_name):

	samples_per_epoch = in_df.shape[0]
	number_of_batches = samples_per_epoch / batch_size
	counter = 0

	while True:

		if batch_size == 0:
			out_df = in_df

		else:
			sub_img_frame = pd.DataFrame(img_df[batch_size * counter:batch_size * (counter + 1)].todense())

			sub_img_frame.columns = ['img_' + str(col) for col in sub_img_frame.columns]

			out_df = in_df[batch_size * counter:batch_size * (counter + 1)]

			for col in sub_img_frame.columns:
				out_df.insert(len(out_df.columns), col, pd.Series(sub_img_frame[col].values, index=out_df.index))

		feed_dict = {col_name: le_encoders[col_name].transform(out_df[col_name].values) for col_name in cat_cols}

		cont_cols = [x for x in out_df.columns if 'img_' in x]
		cont_cols.extend(['price', 'item_seq_number', 'title', 'description'])
		feed_dict['continuous'] = out_df[cont_cols].values

		counter += 1

		yield feed_dict, out_df[loss_name].values

		if counter <= number_of_batches:
			counter = 0


def root_mean_squared_error(y_true, y_pred):
	return K.sqrt(K.mean(K.square(y_pred - y_true)))


def build_model(label_enc_cols):
	all_embeddings, all_inputs = [], []

	for key, val in label_enc_cols.items():
		in_val = Input(shape = (1,), name = key)
		all_embeddings += [Flatten()(Embedding(val.max() + 1, (val.max() + 1) // 2)(in_val))]
		all_inputs += [in_val]

	concat_emb_layer = concatenate(all_embeddings)
	bn_emb = BatchNormalization()(concat_emb_layer)
	emb_layer = Dense(16, activation='relu')(Dropout(0.5)(bn_emb))

	cont_input = Input(shape = (516,), name = 'continuous')
	bn_cont = BatchNormalization()(cont_input)
	cont_feature_layer = Dense(16, activation = 'relu')(Dropout(0.5)(bn_cont))

	full_concat_layer = concatenate([emb_layer, cont_feature_layer])
	full_reduction = Dense(16, activation = 'relu')(full_concat_layer)

	out_layer = Dense(1, activation = 'sigmoid')(full_reduction)
	model = Model(inputs = all_inputs + [cont_input], outputs = [out_layer])

	return model



train_data, test_data, y_col = load_data()
train_data, test_data = fill_na(train_data, test_data)
train_data, test_data = drop_unwanted(train_data, test_data)
train_data, test_data = transform_title_description(train_data, test_data)

all_data = pd.concat([train_data, test_data], sort = False)
le_encoders, label_enc_cols = encode_cat_columns(all_data)

train_data, test_data = scale_num_cols(train_data, test_data)

train_img_features, test_img_features = load_VGG16_img_features()

t_split_df, v_split_df, image_t_split_df, image_v_split_df = split_train_validation(train_data)

model = build_model(label_enc_cols)

optimizer = optimizers.Adam(lr = 0.0005, beta_1 = 0.9, beta_2 = 0.999, epsilon = 0.1, decay = 0.0, amsgrad = False)
model.compile(optimizer = optimizer, loss = root_mean_squared_error, metrics = [root_mean_squared_error])

checkpoint = ModelCheckpoint('best_weights.hdf5', monitor = 'val_loss', verbose = 1, save_best_only = True)
early = EarlyStopping(patience = 2, mode = 'min')

batch_size = 64
model.fit_generator(gen_samples(t_split_df, image_t_split_df, batch_size, y_col),
                    epochs = 1,
                    steps_per_epoch = t_split_df.shape[0] / batch_size,
                    validation_data = next(gen_samples(v_split_df, image_v_split_df, 128, y_col)),
                    validation_steps = 10,
                    callbacks = [checkpoint, early])

test_vars, test_id = next(gen_samples(test_data, test_img_features, test_data.shape[0], loss_name = ''))
model.load_weights('best_weights.hdf5')
preds = model.predict(test_vars)

subm = pd.read_csv("../input/avito-demand-prediction/sample_submission.csv")
subm['deal_probability'] = preds
subm.to_csv('submission_adam.csv', index=False)

```
