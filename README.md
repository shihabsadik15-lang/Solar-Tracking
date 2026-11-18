function Solar_Tracking_PV_Pro
%% EFFICIENCY IMPROVEMENT OF SOLAR PV CELL BY SUN TRACKING METHOD (IMPROVED)
% MATLAB R2015a compatible. No toolboxes needed.
%
% What this version adds over the basic script:
%   - Professional dashboard figure (irradiance, power, energy, efficiency)
%   - Tracker angle plots (what the motors actually have to do)
%   - I-V and P-V curves (single-diode model) at 09:00 and 12:00
%   - Full-year comparison using 12 representative days (monthly gain)
%   - Seasonal ambient temperature, tracker motor energy included
%   - Results automatically saved as PNG images + CSV table in 'Results' folder
%
% Save this file as Solar_Tracking_PV_Pro.m and run:  Solar_Tracking_PV_Pro

clc; close all;

%% ------------------------- USER INPUTS ---------------------------------
par.lat     = 25;      % Latitude (deg, north +)               <-- CHANGE
par.nday    = 172;     % Day of year for daily plots (172 = 21 June)
par.A       = 1.6;     % Panel area (m^2)
par.eta_ref = 0.17;    % Efficiency at STC
par.gammaT  = -0.004;  % Power temperature coefficient (1/C)
par.NOCT    = 45;      % Nominal operating cell temperature (C)
par.Tavg    = 28;      % Yearly average ambient temperature (C)
par.Tseason = 8;       % Seasonal temperature swing (C)
par.Tamp    = 6;       % Day-night temperature swing (C)
par.motor1  = 5;       % Single-axis motor power (W)
par.motor2  = 10;      % Dual-axis motor power (W)
par.duty    = 0.05;    % Fraction of daylight the motors run

% Single-diode module data (60-cell, ~270 W class module)
par.Isc    = 8.2;      % Short-circuit current at STC (A)
par.Voc    = 37;       % Open-circuit voltage at STC (V)
par.Ns     = 60;       % Cells in series
par.a      = 1.3;      % Diode ideality factor
par.Eg     = 1.12;     % Silicon band gap (eV)
par.alphaI = 0.0005;   % Isc temperature coefficient (1/C)

outDir = 'Results';
if ~exist(outDir,'dir'), mkdir(outDir); end

cF = [0.25 0.25 0.25];   % Fixed colour
cS = [0.00 0.45 0.74];   % Single-axis colour
cD = [0.85 0.33 0.10];   % Dual-axis colour
cMat = [cF; cS; cD];     % All colours in one matrix

%% ------------------------- DAILY SIMULATION ----------------------------
r  = pvday(par.nday, par);
t  = r.t;
xl = [floor(min(t(r.sunup)))-1, ceil(max(t(r.sunup)))+1];

gain_s = 100*(r.Enet_sgl  - r.E_fix)/r.E_fix;
gain_d = 100*(r.Enet_dual - r.E_fix)/r.E_fix;

%% ------------------------- ANNUAL SIMULATION ---------------------------
days = [17 47 75 105 135 162 198 228 258 288 318 344];  % Mid-month days
dim  = [31 28 31 30 31 30 31 31 30 31 30 31];           % Days per month
Em = zeros(12,3);
for m = 1:12
    q = pvday(days(m), par);
    Em(m,:) = [q.E_fix, q.Enet_sgl, q.Enet_dual];       % kWh/day
end
Eyear = sum(Em .* repmat(dim',1,3), 1);                 % kWh/year
gainM_s = 100*(Em(:,2)-Em(:,1))./Em(:,1);
gainM_d = 100*(Em(:,3)-Em(:,1))./Em(:,1);
gainY_s = 100*(Eyear(2)-Eyear(1))/Eyear(1);
gainY_d = 100*(Eyear(3)-Eyear(1))/Eyear(1);

%% ------------------------- FIGURE 1: DASHBOARD -------------------------
f1 = figure('Name','Dashboard','NumberTitle','off', ...
            'Position',[50 50 1250 800],'Color','w');

subplot(2,2,1); hold on; grid on; box on;
h1 = plot(t,r.G_fix ,'Color',cF,'LineWidth',2);
h2 = plot(t,r.G_sgl ,'Color',cS,'LineWidth',2);
h3 = plot(t,r.G_dual,'Color',cD,'LineWidth',2);
xlabel('Time of day (h)'); ylabel('Irradiance (W/m^2)');
title('Irradiance Received by Panel'); xlim(xl);
legend([h1 h2 h3],{'Fixed','Single-axis','Dual-axis'},'Location','north');

subplot(2,2,2); hold on; grid on; box on;
fill([t; flipud(t)],[r.P_fix; flipud(r.P_dual)],cD, ...
     'FaceAlpha',0.15,'EdgeColor','none');
h1 = plot(t,r.P_fix ,'Color',cF,'LineWidth',2);
h2 = plot(t,r.P_sgl ,'Color',cS,'LineWidth',2);
h3 = plot(t,r.P_dual,'Color',cD,'LineWidth',2);
xlabel('Time of day (h)'); ylabel('Output power (W)');
title('PV Output Power (shaded = tracking gain)'); xlim(xl);
legend([h1 h2 h3],{'Fixed','Single-axis','Dual-axis'},'Location','north');

subplot(2,2,3); hold on; grid on; box on;
h1 = plot(t,cumtrapz(t,r.P_fix )/1000,'Color',cF,'LineWidth',2);
h2 = plot(t,cumtrapz(t,r.P_sgl )/1000,'Color',cS,'LineWidth',2);
h3 = plot(t,cumtrapz(t,r.P_dual)/1000,'Color',cD,'LineWidth',2);
xlabel('Time of day (h)'); ylabel('Energy (kWh)');
title('Cumulative Energy Generated'); xlim([0 24]);
legend([h1 h2 h3],{'Fixed','Single-axis','Dual-axis'},'Location','northwest');
text(13.5,0.15*r.E_dual, sprintf('Net gain vs fixed:\n Single: +%.1f %%\n Dual: +%.1f %%', ...
     gain_s,gain_d),'BackgroundColor','w','EdgeColor',[0.5 0.5 0.5]);

subplot(2,2,4); hold on; grid on; box on;
h1 = plot(t,r.eff_fix ,'Color',cF,'LineWidth',2);
h2 = plot(t,r.eff_sgl ,'Color',cS,'LineWidth',2);
h3 = plot(t,r.eff_dual,'Color',cD,'LineWidth',2);
xlabel('Time of day (h)'); ylabel('Efficiency (%)');
title('Conversion Efficiency (includes cell temperature loss)');
xlim(xl); ylim([10 18]);
legend([h1 h2 h3],{'Fixed','Single-axis','Dual-axis'},'Location','south');

print(f1,'-dpng','-r200',fullfile(outDir,'1_Dashboard.png'));

%% ------------------------- FIGURE 2: SUN & TRACKER ANGLES --------------
f2 = figure('Name','Angles','NumberTitle','off', ...
            'Position',[80 80 1150 700],'Color','w');
su_ = r.sunup;

subplot(2,2,1); grid on; box on;
plot(t(su_), r.alt(su_)/(pi/180),'Color',cD,'LineWidth',2);
xlabel('Time (h)'); ylabel('Altitude (deg)'); title('Solar Altitude'); xlim(xl);

subplot(2,2,2); grid on; box on;
plot(t(su_), r.az(su_)/(pi/180),'Color',cS,'LineWidth',2);
xlabel('Time (h)'); ylabel('Azimuth (deg)');
title('Solar Azimuth (0 = South, + = West)'); xlim(xl);

subplot(2,2,3); grid on; box on; hold on;
plot(t(su_), r.R(su_),'Color',cS,'LineWidth',2);
xlabel('Time (h)'); ylabel('Rotation angle (deg)');
title('Single-axis Tracker Rotation (E-W)'); xlim(xl);

subplot(2,2,4); grid on; box on; hold on;
plot(t(su_), r.cos_fix(su_) ,'Color',cF,'LineWidth',2);
plot(t(su_), r.cos_sgl(su_) ,'Color',cS,'LineWidth',2);
plot(t(su_), r.cos_dual(su_),'Color',cD,'LineWidth',2);
xlabel('Time (h)'); ylabel('cos(\theta)');
title('Cosine of Incidence Angle (1 = perfectly facing sun)');
legend('Fixed','Single-axis','Dual-axis','Location','south'); xlim(xl); ylim([0 1.05]);

print(f2,'-dpng','-r200',fullfile(outDir,'2_Angles.png'));

%% ------------------------- FIGURE 3: I-V AND P-V CURVES ----------------
f3 = figure('Name','IV PV curves','NumberTitle','off', ...
            'Position',[110 60 1200 750],'Color','w');
hrs = [9 12];
for k = 1:2
    ix = round(hrs(k)/0.05) + 1;
    G  = [r.G_fix(ix)  r.G_sgl(ix)  r.G_dual(ix)];
    Tc = [r.Tc_fix(ix) r.Tc_sgl(ix) r.Tc_dual(ix)];
    cols = {cF,cS,cD};
    subplot(2,2,2*k-1); hold on; grid on; box on;
    subplot(2,2,2*k);   hold on; grid on; box on;
    for s = 1:3
        [V,I,P] = ivcurve(G(s),Tc(s),par);
        [Pm,im] = max(P);
        subplot(2,2,2*k-1);
        hh(s) = plot(V,I,'Color',cols{s},'LineWidth',2);
        plot(V(im),I(im),'o','Color',cols{s},'MarkerFaceColor',cols{s});
        subplot(2,2,2*k);
        plot(V,P,'Color',cols{s},'LineWidth',2);
        plot(V(im),Pm,'o','Color',cols{s},'MarkerFaceColor',cols{s});
    end
    subplot(2,2,2*k-1);
    xlabel('Voltage (V)'); ylabel('Current (A)');
    title(sprintf('I-V Curve at %02d:00 (dots = max power point)',hrs(k)));
    legend(hh,{'Fixed','Single-axis','Dual-axis'},'Location','southwest');
    ylim([0 10]); xlim([0 45]);
    subplot(2,2,2*k);
    xlabel('Voltage (V)'); ylabel('Power (W)');
    title(sprintf('P-V Curve at %02d:00',hrs(k)));
    xlim([0 45]);
end
print(f3,'-dpng','-r200',fullfile(outDir,'3_IV_PV_Curves.png'));

%% ------------------------- FIGURE 4: ANNUAL COMPARISON -----------------
f4 = figure('Name','Annual','NumberTitle','off', ...
            'Position',[140 60 1200 700],'Color','w');
mn = {'Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'};

subplot(2,1,1); box on; grid on;
b = bar(Em,'grouped');
set(b(1),'FaceColor',cF); set(b(2),'FaceColor',cS); set(b(3),'FaceColor',cD);
set(gca,'XTickLabel',mn);
ylabel('Net energy (kWh/day)');
title('Daily Energy in Each Month');
legend('Fixed','Single-axis','Dual-axis','Location','northeastoutside');

subplot(2,1,2); hold on; box on; grid on;
plot(1:12,gainM_s,'-o','Color',cS,'LineWidth',2,'MarkerFaceColor',cS);
plot(1:12,gainM_d,'-s','Color',cD,'LineWidth',2,'MarkerFaceColor',cD);
set(gca,'XTick',1:12,'XTickLabel',mn); xlim([0.5 12.5]);
ylabel('Net gain over fixed (%)');
title(sprintf('Monthly Gain  (Yearly: single +%.1f %%, dual +%.1f %%)',gainY_s,gainY_d));
legend('Single-axis','Dual-axis','Location','northeastoutside');

print(f4,'-dpng','-r200',fullfile(outDir,'4_Annual_Comparison.png'));

%% ------------------------- FIGURE 5: SUMMARY BARS ----------------------
f5 = figure('Name','Summary','NumberTitle','off', ...
            'Position',[170 80 1000 450],'Color','w');
subplot(1,2,1); hold on; box on; grid on;
v = [r.E_fix r.Enet_sgl r.Enet_dual];
for i = 1:3
    bar(i,v(i),0.6,'FaceColor',cMat(i,:));
    text(i,v(i),sprintf('%.2f',v(i)),'HorizontalAlignment','center', ...
         'VerticalAlignment','bottom','FontWeight','bold');
end
set(gca,'XTick',1:3,'XTickLabel',{'Fixed','Single','Dual'});
ylabel('kWh/day'); title('Net Daily Energy'); ylim([0 1.15*max(v)]);

subplot(1,2,2); hold on; box on; grid on;
v = Eyear;
for i = 1:3
    bar(i,v(i),0.6,'FaceColor',cMat(i,:));
    text(i,v(i),sprintf('%.0f',v(i)),'HorizontalAlignment','center', ...
         'VerticalAlignment','bottom','FontWeight','bold');
end
set(gca,'XTick',1:3,'XTickLabel',{'Fixed','Single','Dual'});
ylabel('kWh/year'); title('Net Yearly Energy'); ylim([0 1.15*max(v)]);

print(f5,'-dpng','-r200',fullfile(outDir,'5_Summary.png'));

%% ------------------------- PRINT + SAVE RESULTS ------------------------
fprintf('\n=================================================================\n');
fprintf('  SUN TRACKING PV RESULTS   (Lat = %.1f deg, Day %d)\n', par.lat, par.nday);
fprintf('=================================================================\n');
fprintf('  %-26s %10s %12s %10s\n','','Fixed','Single-axis','Dual-axis');
fprintf('  %-26s %10.1f %12.1f %10.1f\n','Peak power (W)', ...
        max(r.P_fix),max(r.P_sgl),max(r.P_dual));
fprintf('  %-26s %10.3f %12.3f %10.3f\n','Daily energy (kWh)', ...
        r.E_fix,r.E_sgl,r.E_dual);
fprintf('  %-26s %10.3f %12.3f %10.3f\n','Motor energy (kWh)', ...
        0,r.Emotor_sgl,r.Emotor_dual);
fprintf('  %-26s %10.3f %12.3f %10.3f\n','Net daily energy (kWh)', ...
        r.E_fix,r.Enet_sgl,r.Enet_dual);
fprintf('  %-26s %10s %11.2f%% %9.2f%%\n','Net daily gain','-',gain_s,gain_d);
fprintf('  %-26s %10.0f %12.0f %10.0f\n','Net yearly energy (kWh)', ...
        Eyear(1),Eyear(2),Eyear(3));
fprintf('  %-26s %10s %11.2f%% %9.2f%%\n','Net yearly gain','-',gainY_s,gainY_d);
fprintf('=================================================================\n');
fprintf('  Figures and CSV saved in folder: %s\n\n', fullfile(pwd,outDir));

T = table(mn', Em(:,1), Em(:,2), Em(:,3), gainM_s, gainM_d, ...
    'VariableNames',{'Month','Fixed_kWh_day','Single_kWh_day','Dual_kWh_day', ...
                     'Single_gain_pct','Dual_gain_pct'});
writetable(T, fullfile(outDir,'Monthly_Results.csv'));
disp(T);
end

%% ========================= LOCAL FUNCTIONS ==============================
function r = pvday(n, par)
% Simulates one day (day-of-year n) for fixed, single-axis and dual-axis PV.
d2r = pi/180;
t   = (0:0.05:24)';
phi = par.lat*d2r;

% --- Solar geometry ---
delta   = 23.45*sin(2*pi*(284+n)/365)*d2r;
w       = 15*(t-12)*d2r;
sin_alt = sin(phi)*sin(delta) + cos(phi)*cos(delta)*cos(w);
alt     = asin(sin_alt);
sunup   = alt > 0;
zen     = pi/2 - alt;
cos_az  = (sin_alt*sin(phi) - sin(delta)) ./ (cos(alt)*cos(phi));
cos_az  = max(min(cos_az,1),-1);
az      = (2*(w>=0)-1) .* acos(cos_az);

se = -sin(az).*cos(alt);   % East component of sun vector
sn = -cos(az).*cos(alt);   % North component
su =  sin(alt);            % Up component

% --- Panel orientations ---
beta     = par.lat*d2r;                          % fixed tilt = latitude
cos_fix  = max(sn*(-sin(beta)) + su*cos(beta),0).*sunup;
cos_sgl  = sqrt(se.^2 + su.^2).*sunup;           % N-S axis, E-W tracking
cos_dual = double(sunup);
tilt_fix = beta*ones(size(t));
tilt_sgl = abs(atan2(se,su));
tilt_dual= zen;
R        = atan2(se,su)/d2r;                     % single-axis rotation (deg)

% --- Clear-sky irradiance ---
I0  = 1353*(1+0.033*cos(2*pi*n/365));
AM  = zeros(size(t));  AM(sunup) = 1./sin_alt(sunup);
DNI = zeros(size(t));  DNI(sunup) = I0*0.7.^(AM(sunup).^0.678);
DHI = 0.1*DNI.*max(su,0);

G_fix  = DNI.*cos_fix  + DHI.*(1+cos(tilt_fix ))/2;
G_sgl  = DNI.*cos_sgl  + DHI.*(1+cos(tilt_sgl ))/2;
G_dual = DNI.*cos_dual + DHI.*(1+cos(tilt_dual))/2;

% --- Temperature and PV power ---
Tm  = par.Tavg + par.Tseason*sin(2*pi*(n-100)/365);   % seasonal mean
Ta  = Tm + par.Tamp*cos(2*pi*(t-15)/24);
Tc_fix  = Ta + (par.NOCT-20)/800*G_fix;
Tc_sgl  = Ta + (par.NOCT-20)/800*G_sgl;
Tc_dual = Ta + (par.NOCT-20)/800*G_dual;

P_fix  = par.A*par.eta_ref*G_fix .*(1+par.gammaT*(Tc_fix -25));
P_sgl  = par.A*par.eta_ref*G_sgl .*(1+par.gammaT*(Tc_sgl -25));
P_dual = par.A*par.eta_ref*G_dual.*(1+par.gammaT*(Tc_dual-25));

eff_fix = zeros(size(t)); eff_sgl = eff_fix; eff_dual = eff_fix;
k = G_fix  > 20; eff_fix(k)  = 100*P_fix(k) ./(par.A*G_fix(k));
k = G_sgl  > 20; eff_sgl(k)  = 100*P_sgl(k) ./(par.A*G_sgl(k));
k = G_dual > 20; eff_dual(k) = 100*P_dual(k)./(par.A*G_dual(k));
eff_fix(eff_fix==0)   = NaN;
eff_sgl(eff_sgl==0)   = NaN;
eff_dual(eff_dual==0) = NaN;

% --- Energy ---
r.E_fix  = trapz(t,P_fix)/1000;
r.E_sgl  = trapz(t,P_sgl)/1000;
r.E_dual = trapz(t,P_dual)/1000;
sunH = sum(sunup)*0.05;
r.Emotor_sgl  = par.motor1*par.duty*sunH/1000;
r.Emotor_dual = par.motor2*par.duty*sunH/1000;
r.Enet_sgl  = r.E_sgl  - r.Emotor_sgl;
r.Enet_dual = r.E_dual - r.Emotor_dual;

% --- Store for plotting ---
r.t=t; r.sunup=sunup; r.alt=alt; r.az=az; r.R=R;
r.cos_fix=cos_fix; r.cos_sgl=cos_sgl; r.cos_dual=cos_dual;
r.G_fix=G_fix; r.G_sgl=G_sgl; r.G_dual=G_dual;
r.Tc_fix=Tc_fix; r.Tc_sgl=Tc_sgl; r.Tc_dual=Tc_dual;
r.P_fix=P_fix; r.P_sgl=P_sgl; r.P_dual=P_dual;
r.eff_fix=eff_fix; r.eff_sgl=eff_sgl; r.eff_dual=eff_dual;
end

function [V,I,P] = ivcurve(G,Tc,par)
% Single-diode I-V curve of the module for irradiance G (W/m^2), cell temp Tc (C)
V = linspace(0,45,600)';
if G < 1
    I = zeros(size(V)); P = I; return;
end
kB = 1.380649e-23; q = 1.602177e-19;
Tk = Tc + 273.15;  Tr = 298.15;
Vt    = kB*Tk/q;
I0ref = par.Isc/(exp(par.Voc/(par.a*par.Ns*kB*Tr/q))-1);
I0    = I0ref*(Tk/Tr)^3*exp(par.Eg*q/(par.a*kB)*(1/Tr-1/Tk));
Iph   = par.Isc*(G/1000)*(1+par.alphaI*(Tc-25));
I = Iph - I0*(exp(V/(par.a*par.Ns*Vt))-1);
I = max(I,0);
P = V.*I;
end
