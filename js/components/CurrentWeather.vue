<style>
	.current-weather-card {
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
		border-radius: 12px;
		padding: 2rem;
		margin-bottom: 1.5rem;
	}

	.weather-icon {
		font-size: 4rem;
		margin-bottom: 0.5rem;
	}

	.weather-description {
		font-size: 1.5rem;
		font-weight: bold;
		margin-bottom: 0.5rem;
	}

	.current-time {
		font-size: 1.1rem;
		opacity: 0.9;
		margin-bottom: 1rem;
	}

	.wind-display {
		background: rgba(255, 255, 255, 0.2);
		border-radius: 8px;
		padding: 1rem;
		margin-bottom: 1.5rem;
	}

	.wind-icon {
		font-size: 2rem;
		margin-right: 0.5rem;
	}

	.wind-value {
		font-size: 1.3rem;
		font-weight: bold;
	}

	.forecast-hours {
		display: flex;
		gap: 1rem;
		flex-wrap: wrap;
	}

	.forecast-hour {
		background: rgba(255, 255, 255, 0.15);
		border-radius: 8px;
		padding: 1rem;
		min-width: 100px;
		flex: 1;
		text-align: center;
	}

	.forecast-time {
		font-weight: bold;
		margin-bottom: 0.5rem;
	}

	.forecast-weather {
		font-size: 1.5rem;
		margin-bottom: 0.25rem;
	}

	.forecast-label {
		font-size: 0.9rem;
		opacity: 0.9;
	}

	.no-island-warning {
		background: #fff3cd;
		border: 1px solid #ffc107;
		border-radius: 8px;
		padding: 1.5rem;
		color: #856404;
	}
</style>

<template>
	<div>
		<div v-if='!hasIsland' class='no-island-warning'>
			<h4>{{ $t('cNoIsland') }}</h4>
			<p>{{ $t('cNoIslandDesc') }}</p>
		</div>

		<div v-else>
			<div class='current-weather-card'>
				<div class='current-time'>{{ currentTimeFormatted }}</div>
				<div class='weather-icon'>{{ currentWeatherIcon }}</div>
				<div class='weather-description'>{{ currentWeatherDescription }}</div>
			</div>

			<div class='wind-display'>
				<div class='d-flex align-items-center'>
					<span class='wind-icon'>🍃</span>
					<div>
						<div>{{ $t('cWindSpeed') }}</div>
						<div class='wind-value'>{{ currentWindSpeed }}</div>
					</div>
				</div>
			</div>

			<h5>{{ $t('cNextHours') }}</h5>
			<div class='forecast-hours'>
				<div v-for='(hour, index) in nextHours' :key='index' class='forecast-hour'>
					<div class='forecast-time'>{{ hour.time }}</div>
					<div class='forecast-weather'>{{ hour.icon }}</div>
					<div class='forecast-label'>{{ hour.label }}</div>
				</div>
			</div>
		</div>
	</div>
</template>

<script lang='ts'>
import { Vue, Component, Prop } from 'vue-property-decorator'
import { Forecast, DayForecast, Weather, Hemisphere } from '../model'

@Component
export default class CurrentWeather extends Vue {
	@Prop(Forecast) readonly forecast!: Forecast

	get hasIsland(): boolean {
		return this.forecast.island.name !== 'Anyisle'
	}

	get currentTime(): Date {
		const now = new Date()
		now.setTime(now.getTime() + this.forecast.island.offsetMinutes * 60_000)
		return now
	}

	get currentTimeFormatted(): string {
		return this.$d(this.currentTime, 'dateTimeHMS')
	}

	get currentHour(): number {
		return this.currentTime.getHours()
	}

	get currentMinute(): number {
		return this.currentTime.getMinutes()
	}

	get currentDayForecast(): DayForecast | null {
		const today = this.forecast.todayDate
		const year = today.getFullYear()
		const month = today.getMonth() + 1
		const day = today.getDate()
		
		const monthForecast = this.forecast.monthForecasts[month - 1]
		if (!monthForecast) return null
		
		return monthForecast.days[day - 1] || null
	}

	get currentWeatherIcon(): string {
		const dayForecast = this.currentDayForecast
		if (!dayForecast) return '❓'
		
		const hour = this.currentHour
		const weather = dayForecast.weather[hour]
		
		switch (weather) {
			case Weather.Clear: return '☀️'
			case Weather.Sunny: return '🌤️'
			case Weather.Cloudy: return '🌥️'
			case Weather.RainClouds: return '☁️'
			case Weather.Rain: return '🌧️'
			case Weather.HeavyRain: return '⛈️'
			default: return '❓'
		}
	}

	get currentWeatherDescription(): string {
		const dayForecast = this.currentDayForecast
		if (!dayForecast) return this.$t('cUnknown') as string
		
		const hour = this.currentHour
		const minute = this.currentMinute
		const nextHour = (hour + 1) % 24
		
		const currentWeather = dayForecast.weather[hour]
		const nextWeather = dayForecast.weather[nextHour]
		
		// Calculate progress through the hour (0.0 to 1.0)
		const progress = minute / 60
		
		// If we're in the first half of the hour, current weather dominates
		// If we're in the second half, next weather dominates
		if (progress < 0.5) {
			return this.getWeatherDescription(currentWeather, nextWeather, false)
		} else {
			return this.getWeatherDescription(currentWeather, nextWeather, true)
		}
	}

	getWeatherDescription(current: Weather, next: Weather, secondHalf: boolean): string {
		// If weather is the same, just return the weather type
		if (current === next) {
			return this.getWeatherLabel(current)
		}
		
		// Special cases for transitions
		if (secondHalf) {
			// Second half of hour - transitioning to next weather
			if (current === Weather.Clear && next === Weather.Sunny) {
				return this.$t('cFewClouds') as string
			}
			if (current === Weather.Sunny && next === Weather.Clear) {
				return this.$t('cFewClouds') as string
			}
			if ((current === Weather.Clear || current === Weather.Sunny) && 
			    (next === Weather.Rain || next === Weather.HeavyRain)) {
				return this.$t('cThunderstormsNearby') as string
			}
			if ((current === Weather.Rain || current === Weather.HeavyRain) && 
			    (next === Weather.Clear || next === Weather.Sunny)) {
				return this.$t('cClearing') as string
			}
			if (current === Weather.Cloudy && (next === Weather.Clear || next === Weather.Sunny)) {
				return this.$t('cClearing') as string
			}
			if ((current === Weather.Clear || current === Weather.Sunny) && next === Weather.Cloudy) {
				return this.$t('cCloudingOver') as string
			}
			if (current === Weather.Cloudy && (next === Weather.Rain || next === Weather.HeavyRain)) {
				return this.$t('cRainApproaching') as string
			}
			if ((current === Weather.Rain || current === Weather.HeavyRain) && next === Weather.Cloudy) {
				return this.$t('cRainEasing') as string
			}
		} else {
			// First half of hour - current weather still dominant
			return this.getWeatherLabel(current)
		}
		
		// Default to current weather
		return this.getWeatherLabel(current)
	}

	getWeatherLabel(weather: Weather): string {
		switch (weather) {
			case Weather.Clear: return this.$t('cClear') as string
			case Weather.Sunny: return this.$t('cSunny') as string
			case Weather.Cloudy: return this.$t('cCloudy') as string
			case Weather.RainClouds: return this.$t('cRainClouds') as string
			case Weather.Rain: return this.$t('cRain') as string
			case Weather.HeavyRain: return this.$t('cHeavyRain') as string
			default: return this.$t('cUnknown') as string
		}
	}

	get currentWindSpeed(): string {
		const dayForecast = this.currentDayForecast
		if (!dayForecast) return this.$t('cUnknown') as string
		
		const hour = this.currentHour
		const windPower = dayForecast.windPower[hour]
		
		return this.$t('cWindValue', { value: windPower }) as string
	}

	get nextHours(): Array<{time: string, icon: string, label: string}> {
		const dayForecast = this.currentDayForecast
		if (!dayForecast) return []
		
		const result = []
		const startHour = this.currentHour
		
		for (let i = 1; i <= 3; i++) {
			const hour = (startHour + i) % 24
			const weather = dayForecast.weather[hour]
			
			// Create date for this hour
			const hourDate = new Date(this.currentTime)
			hourDate.setHours(hour)
			if (hour < startHour) {
				hourDate.setDate(hourDate.getDate() + 1)
			}
			
			result.push({
				time: this.$d(hourDate, 'timeH'),
				icon: this.getWeatherIcon(weather),
				label: this.getWeatherLabel(weather)
			})
		}
		
		return result
	}

	getWeatherIcon(weather: Weather): string {
		switch (weather) {
			case Weather.Clear: return '☀️'
			case Weather.Sunny: return '🌤️'
			case Weather.Cloudy: return '🌥️'
			case Weather.RainClouds: return '☁️'
			case Weather.Rain: return '🌧️'
			case Weather.HeavyRain: return '⛈️'
			default: return '❓'
		}
	}
}
</script>
