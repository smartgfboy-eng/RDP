name: RDP

on:
  workflow_dispatch:

jobs:
  secure-rdp:
    runs-on: windows-latest
    timeout-minutes: 360

    steps:
      - name: Configure Core RDP Settings
        shell: powershell
        run: |
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
            -Name "fDenyTSConnections" -Value 0 -Force

          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
            -Name "UserAuthentication" -Value 0 -Force

          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
            -Name "SecurityLayer" -Value 0 -Force

          Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

          Restart-Service -Name TermService -Force

      - name: Create RDP User
        shell: powershell
        run: |
          $password = "StrongPass123!"

          $securePass = ConvertTo-SecureString $password -AsPlainText -Force

          if (-not (Get-LocalUser -Name "RDP" -ErrorAction SilentlyContinue)) {
            New-LocalUser -Name "RDP" -Password $securePass -AccountNeverExpires
          }

          Add-LocalGroupMember -Group "Administrators" -Member "RDP" -ErrorAction SilentlyContinue
          Add-LocalGroupMember -Group "Remote Desktop Users" -Member "RDP" -ErrorAction SilentlyContinue

          echo "RDP_USER=RDP" >> $env:GITHUB_ENV
          echo "RDP_PASS=$password" >> $env:GITHUB_ENV

      - name: Install Tailscale
        shell: powershell
        run: |
          Invoke-WebRequest https://pkgs.tailscale.com/stable/tailscale-setup-latest-amd64.msi -OutFile tailscale.msi

          Start-Process msiexec.exe -ArgumentList "/i tailscale.msi /quiet /norestart" -Wait

      - name: Connect Tailscale
        shell: powershell
        run: |
          & "$env:ProgramFiles\Tailscale\tailscale.exe" up --authkey=${{ secrets.TAILSCALE_AUTH_KEY }} --hostname=github-rdp

          Start-Sleep -Seconds 15

          $ip = & "$env:ProgramFiles\Tailscale\tailscale.exe" ip -4

          echo "TS_IP=$ip" >> $env:GITHUB_ENV

      - name: Show RDP Details
        shell: powershell
        run: |
          Write-Host "======================="
          Write-Host "RDP READY"
          Write-Host "IP: $env:TS_IP"
          Write-Host "Username: $env:RDP_USER"
          Write-Host "Password: $env:RDP_PASS"
          Write-Host "======================="

      - name: Keep Alive
        shell: powershell
        run: |
          while ($true) {
            Start-Sleep -Seconds 300
            Write-Host "RDP Still Running..."
          }
