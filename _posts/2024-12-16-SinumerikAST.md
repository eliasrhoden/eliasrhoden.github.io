---
layout: post
title:  "Analysis of Sinumerik servo tuning"
date:   2024-12-16 12:00:00 +0200
categories: sinumerik, control theory
---


I’m very happy that I’m finally able to write this specific blog post, as it has been a long time in the making. Today I will do a case study analysis of the Automatic Servo Tuning (AST) functionality in the Sinumerik CNC system.

Since I first saw that it is possible to save the results from AST into an XML file, I’ve wanted to parse that data and analyze it on a PC outside of the Sinumerik control.

All the code used is avaliable at my Github: https://github.com/eliasrhoden/Sinumerik-Ctrl

# Sinumerik servo control

As of today, Sinumerik uses Sinamics S120 drive system for servo control, and those are based on a cascaded control structure. The current controller is mainly dependent on the motor parameters and usually does not require any application dependent tuning.
The velocity and position loop however do, especially the velocity loop, and this is also the most complicated to tune. The position controller is just a P-controller, so this one is quite easy to tune manually.

But the velocity loop is more complex and is the topic of this blog post. Since the velocity loop is more complex and in order to achieve good tuning without the help of drive experts, Siemens have implemented an auto-tune function. The structure of the velocity loop is a PI controller, where you can enable filters on the measured velocity or filters on the setpoint to the current controller. The filters you can enable are:
* PT1 filters 
* PT2 filters
* General 2nd order filters

With this structure you should be able to create any linear controller or stable transfer function.

For this specific application, the control structure can be simplified substantially. 
This diagram does not contain any feed forward or reference model since they don't influence the stability of the velocity loop.
Please refer to the Siemens manuals if you want a more accurate description.
![](/assets/images/sinumerik_cascaded.drawio.png)


For more information regarding Siemens S120 I would recomend the following manuals:
* Drive Optimization Guide: 
https://support.industry.siemens.com/cs/document/60593549/drive-optimization-guide?dti=0&lc=en-GE 

* S120 List manual:
https://www.industry-mobile-support.siemens-info.com/en/article/detail/109827046

* S120 Function manual:
https://support.industry.siemens.com/cs/document/109781535/sinamics-s120-function-manual-for-drive-functions?dti=0&lc=en-AZ


# Behaviour of AST
I can only guess how the AST works, but based on the types of measurements and structure of the results I would assume that it builds a linear model of the mechanical system, from torque to velocity, and then designs the PI controller and filters based on that plant model.

From my experience with AST it always works well, in those cases where it doesn’t work, it is usually some mechanical issue. In the scenarios where I used AST, the resulting tuning only uses torque setpoint filters, I’ve never seen it use filters on the measured velocity. Which makes sense, theoretically it makes no difference in terms of stability if you filter the controller output, or the measured signal. But filtering the control output would possibly be more robust, since it excites less frequencies of the plant that might contain unknown resonances. 


# Reverse engineering AST Files
As previously said, I’ve been wanting to do this type of analysis for quite some time, and the big hurdle has always been “How do I extract the bode plot from the AST files?”, this section will cover the big picture of that, but for more info I would refer to my github repo Trace-Tools (https://github.com/eliasrhoden/Trace-Tools).

When reading the AST-xml file, one can find a lot of parameters and meta data written in plain text. But the actual measurements or plant models are nowhere to be found. Instead there are a few sections named “FrequencyResponseFunction” that contain large sections of ascii encoded binary data. These are labeled “Base85”, and no joke, I’ve spent way too long trying to figure out how to decode this, I won’t go too much into what base85 if, but it is a technique where you encode 4 bytes into one ascii-character. Thus there is a mapping from int32 -> char. 

```xml
<FrequencyResponseFunction satclassid="6" name="m_pPlant" ver="4.96.0.0.1" env="1">
<NumericArray satclassid="4" elementtype="ComplexNumber" format="base85" name="m_ValueVector" ver="4.96.0.0.1" env="1">
<unsigned_int name="m_length">1078</unsigned_int>
<raw name="m_pNumberArray"><![CDATA[
ZN7RO@z}N~ZOtp@A`u/hkF}hk8H~`wZQsW7ky*C5ZNXv_S=GknZSAooat...
```

If you google around you will quickly find cryptography websites that can convert data to b85 and vice versa, it then becomes evident that Siemens uses a non-standard mapping. Probably in order to be able to use it within xml-files, since some characters in the standard b85 mapping are used in xml-syntax.

I tried many ways to brute force this without success, I read out data points in the Sinumerik control and tried to find it in the data without any luck. Now in hindsight, one reason behind why one of my attempts didn’t work, was that the HMI shows all velocity measurements in mm/min, but in the file, it is encoded in m/s.

Long story short, I found the mapping by looking at the ”FrequencyVector” where I knew that the frequency values are always increasing and by splitting the characters into rows, a clear pattern emerged.

Just to mention it, here is the base85 mapping used:
```
0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!#$`()*+-/:;=?@[]^_{|}~
```

After finding out the mapping, I published TraceTools on my github, where you can extract measurements and frequency responses from AST files and also frequency measurements and step response measurements.


# Control analysis
For the analysis I will compare the AST-controller with a Hinf controller and an IMC controller.
My expectation was that both of these (especially the Hinf) would beat the performance  of the AST controller, but as later will be explained, that was not the case. Unfortunately I will not be able to do any comparison on the real system since I do not have access anymore to the machine where the measurements come from, so this will sadly only be a theoretical comparison.

In the AST file, the response of the current controller is also included, but for simplicity 
(and that I ran into numerical issues trying to simulate the step response of the AST controller) I will not include that in the analysis here. Also the bandwidth of the current controller will not be the limiting factor in this case, as will later be shown.


## Plant model
Starting with the plant, as mentioned this is the velocity control loop, so it is the transfer function from motor torque to motor velocity. This measurement is from a belt driven spindle, where only the motor encoder is available. 

From the bode plot we see that we have an integrating system with one double zero and two double poles.

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/plant_ast.png)

First I select the frequency range of interest and fit a general 3rd order transfer function, it looks good but it contains one unstable zero outside of the measured frequency range. After removing it we end up with a good estimate of the transfer function of the mechanical system.

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/plant_identification.png)

From the identified model, the system has the following zeros and poles 

```
Zeros
PT2:  w0 =  9999.999999999996 zeta =  0.5000000000000003
PT2:  w0 =  266.3612165041341 zeta =  0.04161182525315884
Poles
PT2:  w0 =  10000.0 zeta =  0.04999999999999984
PT2:  w0 =  373.9453600549471 zeta =  0.04113635881529934
Real:  -2.097556148013025
```

where it is evident that the system contains both an undamped double zero and double pole.


## Robust stability
One criteria I will look at later is robust stability, that uses the multiplicative model uncertainty. Given the multiplicative uncertainty, where the *real* plant is denoted $G^*(s)$, and the nominal model is $G(s)$, then the follwoing defines multiplicative uncertainty

$$
W_\Delta (s) = \frac{G^*(s) - G(s)}{G(s)}
$$

Robust stability is then guaranteed if the following holds

$$
|T(j\omega) W_\Delta (j\omega)| < 1 
$$

Where $T$ is the complementary sensitivity function.

In order to determine this I first determine the additive error and then the multiplicative error. In the next figure the true error is showed and the approximation used for analysis.

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/additive_error.png)

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/W_delta.png)

## Hinf controller 
Firstly I’m not an expert in robust control, so my approach might not be the ideal or correct way of doing it so take it with a grain of salt. There exists a lot of material online about Hinf-controllers, but the big picture is that it is an optimization method where the $H_\infty$ norm of a generalized plant is minimized.


The difficult part about Hinf controller synthesis is that one must specify the requirements and signal properties in the frequency domain. In this case, I provide weights/specifications for the Sensitivity function, Complementary sensitivity function and the product of the controller and sensitivity function. 
This is called “Mixed-sensitivity Hinf control design”, as described in this tutorial: https://juliacontrol.github.io/RobustAndOptimalControl.jl/dev/hinf_DC/

I tried to do this in python, but the robust control functions require the installation of slycot that is troublesome on windows so I decided to try Julia for this part of the project, so far I like the robust control toolbox but it will not replace python as my main language.

If you want to dive into the details you can check the Hinf code at: https://github.com/eliasrhoden/Sinumerik-Ctrl/blob/main/hinf_opt.ipynb

## IMC Controller
IMC stands for Internal Model Control, that implies that you have a controller C, that outputs to the plant but also a plant model, and the input to the controller is the error between the plant and model. For nonlinear plants, this results in a quite complicated controller, but for the linear case (with a linear plant) you can simply solve for the controller by inverting the plant. 

So in reality it is no internal model, rather it is plant inversion that is the key, but I will call it IMC since that is how I first discovered this technique.

Given an invertible plant $G(s)$ and a referece model $M(s)$, the controller can be calculated as

$$
C_{imc}(s) = \frac{M(s)}{1 - M(s)} G^{-1}(s)
$$


Since the IMC technique is based on model inversion, it is very important that we only invert stable zeros, since we don’t want any unstable poles in the controller.

The main benefit of IMC is that the tuning consists of picking a reference model (that at least has the same relative degree of the plant). In this case, the plant has a relative degree of one, but I will pick a PT2 system as a reference model since I want to include some overshoot in the closed loop response.

## Controller frequency response

Looking at the bode plots of each controller shows that they are quite similar. The first thing that stands out is that both Hinf and IMC have a resonance frequency at 270 rad/s  while the AST controller does not. I noticed that when trying to achieve higher bandwidth for the Hinf controller, this resonance peak would increase drastically. I assume this resonance comes from trying to cancel the undamped zeros in the plant at that frequency. While I believe this is fine from a theoretical standpoint, I don’t think it’s a robust way to handle it. Not only does the controller excite the plant even further, but if the mechanical properties of the system changes then you have introduced a new resonance which would be undesirable..

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/controllers.png)

## Stability margins

Continuing to compare the stability margins of the open loop, for this comparison I added a Padé delay of 125 us (sample time of the controller) because the plant would not cross 180 degrees for some controllers. 

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/open_loop_margins.png)

We can see that all controllers have somewhat similar margins, I would say that the worst one is Hinf due to its low gain margin. The phase margin of all systems are above 60 degrees which in my opinion is good enough. One possible improvement to the AST would be to include another filter at high frequencies, since that should improve the gain margin without compromising the phase margin, because the 180 degree-crossing is so much higher compared to the Hinf and IMC systems.

## Sensitivity function

Regarding the sensitivity function, they all achieve similar bandwidth, looking at the peak values of all three shows that the IMC is the worst, by lowering the bandwidth of the reference model should reduce this peak, if needed. Other than that the AST controller seems to have better attenuation at lower frequencies as well.
```
S_peak_ast: 1.11
S_peak_hinf: 1.16
S_peak_imc: 1.27
```
![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/sensitivity_functions.png)


## Robust stability

Robust stability
For robust stability, one looks for the highest peak, and here it is interesting to see that the Hinf controller performs the best overall, while the AST performs best in the lower frequency region. The worst one is the IMC controller, and I believe that the reason why the Hinf and IMC performs worse in the 200 rad/s area is due to the resonance in the controller. If a higher filter would have been added for the AST controller, I believe it would perform better in this metric. The high peak is far above the closed loop bandwidth, so another filter at the higher frequencies should not deteriorate the stability margins too much.


![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/robust_stability.png)

One interesting aspect is that Hinf was worse in terms of nominal stability (stability margins of open loop), but better in terms of robust stability.
Looking at the peak values confirms the previous reasoning, all are still robustly stable, since none are above the value 1.

```
T_rob_ast_peak: 0.39
T_rob_hinf_peak: 0.20
T_rob_imc_peak: 0.44
```

## Closed loop

Looking at the closed loop response it is evident that the AST achieves the highest bandwidth, but does have a clear double zero that is less visible in the other controllers, as to be expected, since both the Hinf and IMC tries to cancel the double zero in the plant and AST does not.

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/closed_loop.png)

This compromise is more visible in the step response, where AST has the initial transient overshoot. From both the frequency response and step response, I would say that the IMC clearly looks best, but this is of course at the robust tradeoff with the resonance/double zero cancellation.

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/step_response.png)

The Hinf controller seems to be some middle-ground between them, but I would say that the undamped oscillations after the initial transient is a dealbreaker that this would not work in a real application.

The overshoot of the AST controller is not that important, this is when applying a step input to the velocity reference, and that would never happen in a real scenario, where the NC interpolator would ramp velocity according to the acceleration parameters. Thus is this step response more of an example of how quickly disturbances would be suppressed.

## IMC2 Controller
So far, I would pick IMC in a theoretical scenario but in a real application I would pick the AST controller. But would it be possible to modify the IMC controller to yield a more robust/realistic controller? i.e. without trying to cancel the double zero of the plant?
As one last experiment, I will include the double zero in the reference model and compute a new IMC controller, from here on denoted IMC2.

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/ref_model2.png)


Looking at the bode plot of the controller, it looks more promising, there are no resonances, only zeros. It even has less bandwidth than the AST controller.

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/C_imc2_vs_ast.png)


In terms of robust stability, the IMC2 controller even looks better than the AST controller. They have similar characteristics at low frequencies, but due to the lower magnitude of IMC2 at higher frequencies, we also note that the magnitude of robust stability decreases faster compared to the AST controller.

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/robust_stability_imc2_vs_ast.png)


One drawback of the IMC2 controller is that the sensitivity function has a greater peak compared to the AST system. 

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/sensitivity_imc2_vs_ast.png)

The closed loop also looks very similar to AST, they achieve the same bandwidth, apart from that IMC2 has a faster rolloff than the AST controller.

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/CL_imc2_vs_ast.png)

From the step response it is also evident that the IMC2 controller has a marginally slower rise time, but a shorter settling time and also less overshoot.

![](https://raw.githubusercontent.com/eliasrhoden/Sinumerik-Ctrl/refs/heads/main/figures/step_response_imc2_vs_ast.png)

Considering how similar the IMC2 controller is to the AST and even more conservative in some regards, makes it reasonable to think that it could work on the real system. 
As previously mentioned I no longer have access to the machine that the measurements were obtained from, but maybe it will be possible in the future.


# Conclusion
The effect of the double zeros had a much bigger impact than what I initially thought, also if one would try to increase the bandwidth further of the reference model used for IMC2, the initial “valley” in the transient would increase, that was the main limiting factor when designing IMC2. 

Secondly I’m even more impressed by the performance of the AST, considering I’ve spent many hours on this analysis and design, on a modern PC. While the AST did it’s design in less than 5 minutes, on an industrial embedded system. That is really impressive.

TL;DR: AST Works really well, and the main limitations are the zeros of the system, even if they are stable. You can achieve higher bandwidth but at the cost of less robustness by canceling the undamped zeros in the plant.

If you want to have a look at the code used for this analysis, it can be found on my github: https://github.com/eliasrhoden/Sinumerik-Ctrl

