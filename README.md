# Whole code

import pandas as pd
import numpy as np
from matplotlib import pyplot as plt
import torch
import torch.nn as nn
import torch.nn.functional as F
import torchvision
import torchvision.transforms as transforms
import collections
import random
from torch import optim
from tqdm import tqdm
from torchvision import datasets
from torch.autograd import Variable
from copy import deepcopy
from torchsummary import summary

print(f"Pytorch version: {torch.__version__}")

if not torch.cuda.is_available():
    raise Exception("GPU device not found, runtime environment should be set to GPU")
print(f"Using GPU device: {torch.cuda.get_device_name(torch.cuda.current_device())}")

torch.manual_seed(0)
np.random.seed(0)

device = torch.device("cuda")

# hyperparameters
dataset_name = 'cifar10'
image_size = (3, 32, 32)
batch_size = 128
num_classes = 10
num_tasks = 5
random_splits = False


# Load train and test datasets

train_transforms = transforms.Compose([transforms.ToTensor(),])
test_transforms = transforms.Compose([transforms.ToTensor(),])

augmentation_transforms = [transforms.RandomCrop(32, padding=4),
                           transforms.RandomHorizontalFlip()]

if augmentation_transforms is not None:
    train_transforms = transforms.Compose(augmentation_transforms
                                          + [transforms.ToTensor()])

train_dataset = torchvision.datasets.CIFAR10(root='./data',
                                             train=True,
                                             download=True,
                                             transform=train_transforms)

test_dataset = torchvision.datasets.CIFAR10(root='./data',
                                            train=False,
                                            download=True,
                                            transform=test_transforms)

print(f"Number of train images: {len(train_dataset)}\n", # 50.000
      f"Number of test images: {len(test_dataset)}\n", # 10.000
      f"Number of classes: {len(np.unique(train_dataset.targets))}\n") # 10


# Create dataset loaders, each corresponding to a different task for training
trainset_loaders = []
testset_loaders = []

targets = [train_dataset[i][1] for i in range(len(train_dataset))]
labels = np.unique(targets)

num_concurrent_labels = num_classes // num_tasks

# label subsets (a form of class incremental learning). This means each task involves different classes from the CIFAR10 dataset.
for i in range(0, num_classes, num_concurrent_labels):
    concurrent_labels = labels[i: i + num_concurrent_labels]

    concurrent_targets = np.isin(targets, concurrent_labels)
    filtered_indices = np.where(concurrent_targets)[0]
    train_ds_subset = torch.utils.data.Subset(train_dataset, filtered_indices)

    trainset_loader = torch.utils.data.DataLoader(train_ds_subset,
                                                  batch_size=batch_size,
                                                  shuffle=True,
                                                  num_workers=2)
    trainset_loaders.append(trainset_loader)

# Test subset
    test_targets = [test_dataset[i][1] for i in range(len(test_dataset))]
    concurrent_test_targets = np.isin(test_targets, concurrent_labels)
    filtered_test_indices = np.where(concurrent_test_targets)[0]
    test_ds_subset = torch.utils.data.Subset(test_dataset, filtered_test_indices)

    testset_loader = torch.utils.data.DataLoader(test_ds_subset,
                                                 batch_size=batch_size,
                                                 shuffle=False,
                                                 num_workers=2)
    testset_loaders.append(testset_loader)


# Create new train dataset loaders of random splits if hyperparameter is True
if random_splits:
    trainset_loaders = []
    ds_length = len(train_dataset)
    splits_length = ds_length // num_tasks
    rand = np.random.permutation(len(train_dataset))
    ds_rand_parts = torch.utils.data.random_split(train_dataset,
                                                  [splits_length] * num_tasks,
                                                  generator=torch.Generator().manual_seed(0))
    for item in ds_rand_parts:
        trainset_loader = torch.utils.data.DataLoader(item,
                                                      batch_size=batch_size,
                                                      shuffle=True,
                                                      num_workers=2)
        trainset_loaders.append(trainset_loader)

# Define the BasicBlock
def conv3x3(in_planes, out_planes, stride=1):
    return nn.Conv2d(in_planes, out_planes, kernel_size=3, stride=stride,
                     padding=1, bias=False)

class BasicBlock(nn.Module):
    expansion = 1

    def __init__(self, in_planes, planes, stride=1):
        super(BasicBlock, self).__init__()
        self.conv1 = conv3x3(in_planes, planes, stride)
        self.bn1 = nn.BatchNorm2d(planes)
        self.relu = nn.ReLU(inplace=True)
        self.conv2 = conv3x3(planes, planes)
        self.bn2 = nn.BatchNorm2d(planes)

        self.shortcut = nn.Sequential()
        if stride != 1 or in_planes != planes:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_planes, planes, kernel_size=1,
                          stride=stride, bias=False),
                nn.BatchNorm2d(planes)
            )

    def forward(self, x):
        out = self.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        out += self.shortcut(x)
        out = self.relu(out)
        return out

# Define the ResNet model
class ResNet(nn.Module):
    def __init__(self, block, num_blocks, num_classes):
        super(ResNet, self).__init__()
        self.in_planes = 64
        self.conv1 = nn.Conv2d(3, 64, kernel_size=3, stride=1,
                               padding=1, bias=False)
        self.bn1 = nn.BatchNorm2d(64)
        self.relu = nn.ReLU(inplace=True)
        self.layer1 = self._make_layer(block, 64, num_blocks[0], stride=1)
        self.layer2 = self._make_layer(block, 128, num_blocks[1], stride=2)
        self.layer3 = self._make_layer(block, 256, num_blocks[2], stride=2)
        self.layer4 = self._make_layer(block, 512, num_blocks[3], stride=2)
        self.avgpool = nn.AdaptiveAvgPool2d((1, 1))
        self.fc = nn.Linear(512 * block.expansion, num_classes)

    def _make_layer(self, block, planes, num_blocks, stride):
        strides = [stride] + [1] * (num_blocks - 1)
        layers = []
        for stride in strides:
            layers.append(block(self.in_planes, planes, stride))
            self.in_planes = planes * block.expansion
        return nn.Sequential(*layers)

    def forward(self, x):
        x = self.relu(self.bn1(self.conv1(x)))
        x = self.layer1(x)
        x = self.layer2(x)
        x = self.layer3(x)
        x = self.layer4(x)
        x = self.avgpool(x)
        x = x.view(x.size(0), -1)
        x = self.fc(x)
        return x

model = ResNet(BasicBlock, [2, 2, 2, 2], num_classes).to(device)

# EWC

def variable(t: torch.Tensor, use_cuda=True, **kwargs):
    if torch.cuda.is_available() and use_cuda:
        t = t.cuda()
    return Variable(t, **kwargs)

class EWC(object):
    def __init__(self, model: nn.Module, dataloader: torch.utils.data.DataLoader, device, criterion):

        self.model = model
        self.dataloader = dataloader
        self.device = device
        self.criterion = criterion

        self.params = {n: p for n, p in self.model.named_parameters() if p.requires_grad}
        self._means = {}
        self._precision_matrices = self._diag_fisher()

        for n, p in deepcopy(self.params).items():
            self._means[n] = variable(p.data)

    # Diagonal Fisher matrix
    def _diag_fisher(self):
        precision_matrices = {}
        for n, p in deepcopy(self.params).items():
            p.data.zero_()
            precision_matrices[n] = variable(p.data)

        self.model.eval()
        for inputs, labels in self.dataloader:
            self.model.zero_grad()
            inputs = inputs.to(self.device)
            labels = labels.to(self.device)
            output = self.model(inputs)
            loss = self.criterion(output, labels)
            loss.backward()

            for n, p in self.model.named_parameters():
                if p.grad is not None:
                    precision_matrices[n].data += p.grad.data ** 2 / len(self.dataloader.dataset)

        precision_matrices = {n: p for n, p in precision_matrices.items()}
        return precision_matrices

    def penalty(self, model: nn.Module):
        loss = 0
        for n, p in model.named_parameters():
            _loss = self._precision_matrices[n] * (p - self._means[n]) ** 2
            loss += _loss.sum()
        return loss

class ReplayBuffer:
    def __init__(self, buffer_size, device="cpu"):
        self.buffer_size = buffer_size
        self.device = device
        self.buffer = collections.deque(maxlen=buffer_size)
        self.counter = 0  # Counter to track the total number of samples seen

    def push(self, state, action):
        if len(self.buffer) < self.buffer_size:
            self.buffer.append((state, action))
        else:
            # Insert at a random position if the buffer is full
            idx = random.randint(0, self.counter)
            if idx < self.buffer_size:
                self.buffer[idx] = (state, action)
        self.counter += 1

    def sample(self, batch_size):
        indices = np.random.choice(len(self.buffer), batch_size, replace=False)
        states, actions = zip(*[self.buffer[idx] for idx in indices])
        return states, actions

    def __len__(self):
        return len(self.buffer)

    def is_empty(self):
        return len(self.buffer) == 0

# Initialize Replay Buffer
replay_buffer_capacity = 5120
replay_buffer = ReplayBuffer(replay_buffer_capacity, device='cuda')

# Function to calculate accuracy
def get_accuracy(model_output, target, batch_size):
    prediction = torch.max(model_output, 1)[1].view(target.size())
    corrects = (prediction.data == target.data).sum()
    accuracy = 100.0 * corrects / batch_size
    return accuracy.item()

# Function to evaluate model on a single test loader

def evaluate_model(test_loader, model, loss_function):
    model.eval()
    test_loss = 0.0
    test_acc = 0.0
    for i, (images, labels) in enumerate(test_loader):
        images = images.to(device)
        labels = labels.to(device)
        predictions = model(images)
        loss = loss_function(predictions, labels)
        test_loss += loss.detach().item()
        test_acc += get_accuracy(predictions, labels, batch_size)
    test_loss = test_loss / (i+1)
    test_acc = test_acc / (i+1)
    return test_loss, test_acc

def evaluate_train(train_loader, model, loss_function):
    model.eval()
    train_loss = 0.0
    train_acc = 0.0
    total_samples = 0
    for i, (images, labels) in enumerate(train_loader):
        images = images.to(device)
        labels = labels.to(device)
        predictions = model(images)
        loss = loss_function(predictions, labels)
        train_loss += loss.detach().item() * images.size(0)
        train_acc += (predictions.argmax(1) == labels).sum().item()
        total_samples += labels.size(0)
    train_loss = train_loss / total_samples
    train_acc = train_acc / total_samples * 100
    return train_loss, train_acc

# Parameters
learning_rate = 0.1  
momentum = 0.9
weight_decay = 1e-4
optimizer = torch.optim.SGD(model.parameters(), lr=learning_rate, momentum=momentum, weight_decay=weight_decay)  # SGD with momentum

# Learning Rate Scheduler
scheduler = torch.optim.lr_scheduler.StepLR(optimizer, step_size=10, gamma=0.1)

loss_function = nn.CrossEntropyLoss()
num_epochs = 20 

history = {'train_loss': [],
            'train_accuracy': [],
            'test_loss': [],
            'test_accuracy': []}

train_acc_per_task = {i: [] for i in range(num_tasks)}
test_acc_per_task = {i: [] for i in range(num_tasks)}
train_loss_per_task = {i: [] for i in range(num_tasks)}
test_loss_per_task = {i: [] for i in range(num_tasks)}

print(f"Train the model on CIFAR-10 dataset for {num_tasks} x {num_epochs} epochs...\n")

# Train the model using EWC
lambda_ewc = 0.3 # hiperparameter [1,100] [0, 1] # 0, 0.3, 0.5, 0.7, 1, 5, 10,
ewc_tasks_array = [] # list to hold past task data

for task_id, train_loader in enumerate(trainset_loaders):
    print(f"\nTraining on task: {task_id}\n")

    if task_id == 0:
        ewc_tasks = None

    # If this is not the first task, create EWC from previous tasks
    if task_id > 0:
        # create an EWC object. This will compute Fisher Information matrix and store it inside the object
        ewc = EWC(model, ewc_tasks, device, loss_function)

    for epoch in range(num_epochs):
        model.train()
        total_train_loss = 0.0
        total_train_acc = 0.0
        total_samples = 0

        for i, (images, labels) in enumerate(train_loader):
            images = images.to(device)
            labels = labels.to(device)

            # Forward pass + backprop + loss calculation
            predictions = model(images)
            base_loss = loss_function(predictions, labels)

            # -------------- EWC ------------
            # compute EWC penalty
            ewc_loss = 0.0
            if task_id > 0: # if it's not the first task, include EWC loss
                ewc_loss = ewc.penalty(model)

            # ------------- ER --------------
            # Store experiences in replay buffer
            for img, lbl in zip(images, labels):
                replay_buffer.push(img.cpu().detach(), lbl.cpu().detach())

            # Sample from replay buffer more frequently in initial phases
            replay_ratio = 3 if epoch < num_epochs // 3 else 2 if epoch < 2 * num_epochs // 3 else 1
            replay_loss = 0.0
            if not replay_buffer.is_empty():
                for _ in range(replay_ratio):
                    replay_states, replay_actions = replay_buffer.sample(min(batch_size, len(replay_buffer)))
                    replay_states = torch.stack(replay_states).to(device)
                    replay_actions = torch.stack(replay_actions).to(device)
                    replay_predictions = model(replay_states)
                    replay_loss += loss_function(replay_predictions, replay_actions)

            # Combine base loss and ewc penalty
            total_loss = base_loss + lambda_ewc * ewc_loss + replay_loss

            optimizer.zero_grad()
            total_loss.backward()

            # Gradient clipping for stability
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

            # Update model params
            optimizer.step()

            total_train_loss += total_loss.item() * images.size(0)
            total_train_acc += (predictions.argmax(1) == labels).sum().item()
            total_samples += labels.size(0)

        average_train_loss = total_train_loss / total_samples
        average_train_accuracy = total_train_acc / total_samples * 100

        print(f"Task {task_id + 1} | Epoch {epoch + 1} | Train loss: {average_train_loss:.4f} | Train accuracy: {average_train_accuracy:.2f}%")

        # Evaluate the model on all training sets
        for eval_train_id, eval_train_loader in enumerate(trainset_loaders):
            train_loss, train_acc = evaluate_train(eval_train_loader, model, loss_function)
            print(f"\tEvaluation on train task {eval_train_id + 1} | Train loss: {train_loss:.4f} | Train accuracy: {train_acc:.2f}%")

            # Add results to the history dict
            history['train_loss'].append(train_loss)
            history['train_accuracy'].append(train_acc)

            train_loss_per_task[eval_train_id].append(train_loss)
            train_acc_per_task[eval_train_id].append(train_acc)


        # Evaluate the model on all tasks
        for eval_task_id, eval_loader in enumerate(testset_loaders):
            test_loss, test_acc = evaluate_model(eval_loader, model, loss_function)
            print(f"\tEvaluation on test task {eval_task_id + 1} | Test loss: {test_loss:.4f} | Test accuracy: {test_acc:.2f}%")

            # Add results to the history dict
            history['test_loss'].append(test_loss)
            history['test_accuracy'].append(test_acc)

            test_loss_per_task[eval_task_id].append(test_loss)
            test_acc_per_task[eval_task_id].append(test_acc)

    # After training on each task, save a subset of the task's data for computing EWC in future tasks
    task_sample = [train_loader.dataset[i] for i in range(len(train_loader.dataset)) if i % 10 == 0] # save every 10th sample
    ewc_tasks_array.extend(task_sample)

    ewc_tasks = torch.utils.data.DataLoader(ewc_tasks_array, batch_size=batch_size, shuffle=False)

