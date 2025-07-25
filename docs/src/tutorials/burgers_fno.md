# Burgers Equation using Fourier Neural Operator

## Data Loading

```@example burgers_fno
using DataDeps, MAT, MLUtils
using PythonCall, CondaPkg # For `gdown`
using Printf

const gdown = pyimport("gdown")

register(
    DataDep(
        "Burgers",
        """
        Burgers' equation dataset from
        [fourier_neural_operator](https://github.com/zongyi-li/fourier_neural_operator)

        mapping between initial conditions to the solutions at the last point of time \
        evolution in some function space.

        u(x,0) -> u(x, time_end):

          * `a`: initial conditions u(x,0)
          * `u`: solutions u(x,t_end)
        """,
        "https://drive.google.com/uc?id=16a8od4vidbiNR3WtaBPCSZ0T3moxjhYe",
        "9cbbe5070556c777b1ba3bacd49da5c36ea8ed138ba51b6ee76a24b971066ecd";
        fetch_method=(url, local_dir) -> begin
            pyconvert(String, gdown.download(url, joinpath(local_dir, "Burgers_R10.zip")))
        end,
        post_fetch_method=unpack,
    ),
)

filepath = joinpath(datadep"Burgers", "burgers_data_R10.mat")

const N = 2048
const Δsamples = 2^3
const grid_size = div(2^13, Δsamples)
const T = Float32

file = matopen(filepath)
x_data = reshape(T.(collect(read(file, "a")[1:N, 1:Δsamples:end])), N, :)
y_data = reshape(T.(collect(read(file, "u")[1:N, 1:Δsamples:end])), N, :)
close(file)

x_data = reshape(permutedims(x_data, (2, 1)), grid_size, 1, N);
y_data = reshape(permutedims(y_data, (2, 1)), grid_size, 1, N);
```

## Model

```@example burgers_fno
using Lux, NeuralOperators, Optimisers,  Random, Reactant

const cdev = cpu_device()
const xdev = reactant_device(; force=true)

fno = FourierNeuralOperator(
    (16,), 2, 1, 32; activation=gelu, stabilizer=tanh
)
ps, st = Lux.setup(Random.default_rng(), fno) |> xdev;
```

## Training

```@example burgers_fno
dataloader = DataLoader((x_data, y_data); batchsize=128, shuffle=true) |> xdev;

function train_model!(model, ps, st, dataloader; epochs=1000)
    train_state = Training.TrainState(model, ps, st, Adam(0.0001f0))

    for epoch in 1:epochs
        loss = -Inf
        for data in dataloader
            (_, loss, _, train_state) = Training.single_train_step!(
                AutoEnzyme(), MAELoss(), data, train_state; return_gradients=Val(false)
            )
        end

        if epoch % 100 == 1 || epoch == epochs
            @printf("Epoch %d: loss = %.6e\n", epoch, loss)
        end
    end

    return train_state.parameters, train_state.states
end

ps_trained, st_trained = train_model!(fno, ps, st, dataloader)
nothing #hide
```

## Plotting

```@example burgers_fno
using CairoMakie, AlgebraOfGraphics
const AoG = AlgebraOfGraphics
AoG.set_aog_theme!()

x_data_dev = x_data |> xdev;
y_data_dev = y_data |> xdev;

grid = range(0, 1; length=grid_size)
pred = first(
    Reactant.with_config(;
        convolution_precision=PrecisionConfig.HIGH,
        dot_general_precision=PrecisionConfig.HIGH,
    ) do
        @jit(fno(x_data_dev, ps_trained, st_trained))
    end
) |> cdev

data_sequence, sequence, repeated_grid, label = Float32[], Int[], Float32[], String[]
for i in 1:16
    append!(repeated_grid, repeat(grid, 2))
    append!(sequence, repeat([i], grid_size * 2))
    append!(label, repeat(["Ground Truth"], grid_size))
    append!(label, repeat(["Predictions"], grid_size))
    append!(data_sequence, vec(y_data[:, 1, i]))
    append!(data_sequence, vec(pred[:, 1, i]))
end
plot_data = (; data_sequence, sequence, repeated_grid, label)

draw(
    AoG.data(plot_data) *
    mapping(
        :repeated_grid => L"x",
        :data_sequence => L"u(x)";
        color=:label => "",
        layout=:sequence => nonnumeric,
        linestyle=:label => "",
    ) *
    visual(Lines; linewidth=4),
    scales(; Color=(; palette=:tab10), LineStyle = (; palette = [:solid, :dash]));
    figure=(;
        size=(1024, 1024),
        title="Using FNO to solve the Burgers equation",
        titlesize=25,
    ),
    axis=(; xlabelsize=25, ylabelsize=25),
    legend=(; label=L"u(x)", position=:bottom, labelsize=20),
)
```
