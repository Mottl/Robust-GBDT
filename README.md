# Robust-GBDT
Code for the paper: [Robust-GBDT: GBDT with Nonconvex Loss for Tabular Classification in the Presence of Label Noise and Class Imbalance](https://arxiv.org/pdf/2310.05067.pdf)


## Demo for binary classification using XGBoost

```
import optuna
import xgboost as xgb
from sklearn.metrics import roc_auc_score, average_precision_score, accuracy_score
from sklearn.model_selection import  StratifiedKFold, train_test_split  
import numpy as np
from sklearn.model_selection import cross_val_score
from rfl_loss import RFLBinary, XGBRFLMulti


# define model
def robustxgb_binary(X_train, y_train, X_test, y_test, n_trials=10):
    sampler = optuna.samplers.TPESampler(seed=42)
    study = optuna.create_study(direction="maximize",
                                sampler=sampler,
                                study_name='xgb_eval')
    study.optimize(RobustXGBBinary(X_train, y_train), n_trials=n_trials)

    print("Best parameters:", study.best_trial.params)

    # Train and evaluate the model with the best hyperparameters
    best_params = study.best_trial.params
    model = xgb.XGBClassifier(max_depth=best_params['max_depth'],
                                reg_alpha=best_params['reg_alpha'],
                                reg_lambda=best_params['reg_lambda'],
                                learning_rate=best_params['learning_rate'],
                                n_estimators=best_params['n_estimators'],
                                objective=RFLBinary(best_params['r'], q=best_params['q']),
                                # tree_method=params['tree_method']
                                )
    model.fit(X_train, y_train)
    
    y_pred_proba = model.predict_proba(X_test)[:, 1]  
    auc = roc_auc_score(y_test, y_pred_proba)
    aucpr = average_precision_score(y_test, y_pred_proba)
    print(f'Test AUC: {auc:.4f}')
    print(f'Test AUCPR: {aucpr:.4f}')
    return auc, aucpr


class RobustXGBBinary(object):
    def __init__(self, X, y):

        self.X = X
        self.y = y

    def __call__(self, trial):
        params = {
        'max_depth': trial.suggest_int('max_depth', 2, 10),
        'reg_alpha': trial.suggest_float('reg_alpha', 1e-4, 1.0),
        'reg_lambda': trial.suggest_float('reg_lambda', 1e-4, 5.0),
        'learning_rate': trial.suggest_float('learning_rate', 1e-3, 1.0),
        'n_estimators':trial.suggest_int('n_estimators', 10, 200, 10),
        "r": trial.suggest_categorical("r", [0.0, 0.5, 1.0]),
        "q": trial.suggest_categorical("q", [0.0, 0.1, 0.3, 0.5]),
        # "tree_method": 'gpu_hist'
    }
    
        clf = xgb.XGBClassifier(max_depth=params['max_depth'],
                                reg_alpha=params['reg_alpha'],
                                reg_lambda=params['reg_lambda'],
                                learning_rate=params['learning_rate'],
                                n_estimators=params['n_estimators'],
                                objective=RFLBinary(r=params['r'], q=params['q']),
                                # tree_method=params['tree_method']
                                )
        cv = StratifiedKFold(n_splits=5, random_state=42, shuffle=True)
        auc_scores = cross_val_score(clf, self.X, self.y, cv=cv, scoring='roc_auc')
        return auc_scores.mean()
      

# load data
from sklearn.datasets import load_breast_cancer
data = load_breast_cancer()  
X = data.data.astype(np.float32)
y = data.target
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, random_state=42)


# train model
robustxgb_binary(X_train, y_train, X_test, y_test)
```

## Demo for binary classification using LightGBM
```
import optuna
from sklearn.metrics import roc_auc_score, average_precision_score
from sklearn.model_selection import  StratifiedKFold, train_test_split  
import numpy as np
from rfl_loss import RFLBinary
import lightgbm as lgb


# define some functions
def sigmoid(x):
    kEps = 1e-16 #  avoid 0 div
    x = np.minimum(-x, 88.7)  # avoid exp overflow
    return 1 / (1 + np.exp(x)+kEps)


def predict_proba(model, X):
    # Lightgbm: Cannot compute class probabilities or labels due to the usage of customized objective function.
    prediction = model.predict(X)
    
    prediction_probabilities = sigmoid(prediction).reshape(-1, 1)
    prediction_probabilities = np.concatenate((1 - prediction_probabilities,
                                                    prediction_probabilities), 1)
    return prediction_probabilities

def eval_auc(labels, preds):  # auc
    p = sigmoid(preds)
    return 'auc', roc_auc_score(labels, p), True


# define model
def robustlgb_binary(X_train, y_train, X_test, y_test, n_trials=10):
    optuna.logging.set_verbosity(optuna.logging.WARNING)  
    sampler = optuna.samplers.TPESampler(seed=42)
    study = optuna.create_study(direction="maximize",
                                sampler=sampler,
                                study_name='lgb_eval')
    study.optimize(RobustLGBBinary(X_train, y_train), n_trials=n_trials)

    print("Best parameters:", study.best_trial.params)

    # Train and evaluate the model with the best hyperparameters
    best_params = study.best_trial.params
    model = lgb.LGBMClassifier(num_leaves=best_params['num_leaves'],
                                reg_alpha=best_params['reg_alpha'],
                                reg_lambda=best_params['reg_lambda'],
                                learning_rate=best_params['learning_rate'],
                                n_estimators=best_params['n_estimators'],
                                objective=RFLBinary(best_params['r'], q=best_params['q']),
                                verbose=-1
                                )
    model.fit(X_train, y_train)
    
    # y_pred_proba = model.predict_proba(X_test)[:, 1]  
    y_pred_proba = predict_proba(model, X_test)[:, 1]
    auc = roc_auc_score(y_test, y_pred_proba)
    aucpr = average_precision_score(y_test, y_pred_proba)
    print(f'Test AUC: {auc:.4f}')
    print(f'Test AUCPR: {aucpr:.4f}')
    return auc, aucpr


class RobustLGBBinary(object):
    def __init__(self, X, y):

        self.X = X
        self.y = y

    def __call__(self, trial):
        params = {
        'num_leaves': trial.suggest_int('num_leaves', 2, 10),
        'reg_alpha': trial.suggest_float('reg_alpha', 1e-4, 1.0),
        'reg_lambda': trial.suggest_float('reg_lambda', 1e-4, 5.0),
        'learning_rate': trial.suggest_float('learning_rate', 1e-3, 1.0),
        'n_estimators':trial.suggest_int('n_estimators', 10, 200),
        "r": trial.suggest_categorical("r", [0.0, 0.5, 1.0]),
        "q": trial.suggest_categorical("q", [0.0, 0.1, 0.3, 0.5]),
    }
    

        cv = StratifiedKFold(n_splits=5, random_state=42, shuffle=True)
        auc_scores = []
        
        for train_index, val_index in cv.split(self.X, self.y):
            X_train, y_train = self.X[train_index], self.y[train_index]
            X_val, y_val = self.X[val_index], self.y[val_index]
            
            model = lgb.LGBMClassifier(num_leaves=params['num_leaves'],
                                reg_alpha=params['reg_alpha'],
                                reg_lambda=params['reg_lambda'],
                                learning_rate=params['learning_rate'],
                                n_estimators=params['n_estimators'],
                                objective=RFLBinary(r=params['r'], q=params['q']),
                                verbose=-1
                                )
            
            model.fit(X_train, y_train)
                    # eval_set=[(X_val, y_val)],
                    # eval_metric=eval_auc)
            
            y_val_pred_prob = predict_proba(model, X_val)[:, 1]
            auc = roc_auc_score(y_val, y_val_pred_prob)
            auc_scores.append(auc)
        return np.mean(auc_scores)
      

# load data
from sklearn.datasets import load_breast_cancer
data = load_breast_cancer()  
X = data.data.astype(np.float32)
y = data.target
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, random_state=42)


# train model
robustlgb_binary(X_train, y_train, X_test, y_test)
```


## Demo for multi-class classification


```
# define model
def robustxgb_multi(X_train, y_train, X_test, y_test, n_trials=10):
    sampler = optuna.samplers.TPESampler(seed=42)
    study = optuna.create_study(direction="maximize",
                                sampler=sampler,
                                study_name='xgb_eval')
    study.optimize(RobustXGBMulti(X_train, y_train), n_trials=n_trials)

    print("Best parameters:", study.best_trial.params)

    # Train and evaluate the model with the best hyperparameters
    best_params = study.best_trial.params
    model = xgb.XGBClassifier(max_depth=best_params['max_depth'],
                                reg_alpha=best_params['reg_alpha'],
                                reg_lambda=best_params['reg_lambda'],
                                learning_rate=best_params['learning_rate'],
                                n_estimators=best_params['n_estimators'],
                                objective=XGBRFLMulti(best_params['r'], q=best_params['q']),
                                # tree_method=params['tree_method']
                                )
    model.fit(X_train, y_train)
    
    y_pred_proba = model.predict(X_test)  
    acc = accuracy_score(y_test, y_pred_proba)
    print(f'Test ACC: {acc:.4f}')
    return acc


class RobustXGBMulti(object):
    def __init__(self, X, y):

        self.X = X
        self.y = y

    def __call__(self, trial):
        params = {
        'max_depth': trial.suggest_int('max_depth', 2, 10),
        'reg_alpha': trial.suggest_float('reg_alpha', 1e-4, 1.0),
        'reg_lambda': trial.suggest_float('reg_lambda', 1e-4, 5.0),
        'learning_rate': trial.suggest_float('learning_rate', 1e-3, 1.0),
        'n_estimators':trial.suggest_int('n_estimators', 10, 200, 10),
        "r": trial.suggest_categorical("r", [0.0, 0.5, 1.0]),
        "q": trial.suggest_categorical("q", [0.0, 0.1, 0.3, 0.5]),
        # "tree_method": 'gpu_hist'
    }
    
        clf = xgb.XGBClassifier(max_depth=params['max_depth'],
                                reg_alpha=params['reg_alpha'],
                                reg_lambda=params['reg_lambda'],
                                learning_rate=params['learning_rate'],
                                n_estimators=params['n_estimators'],
                                objective=XGBRFLMulti(r=params['r'], q=params['q']),
                                # tree_method=params['tree_method']
                                )
        cv = StratifiedKFold(n_splits=5, random_state=42, shuffle=True)
        auc_scores = cross_val_score(clf, self.X, self.y, cv=cv, scoring='accuracy')
        return auc_scores.mean()


# load data
from sklearn.datasets import load_iris
data = load_iris()  
X = data.data.astype(np.float32)
y = data.target
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, random_state=42)


# train model
robustxgb_multi(X_train, y_train, X_test, y_test)
